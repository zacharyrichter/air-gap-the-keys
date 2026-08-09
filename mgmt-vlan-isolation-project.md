# Building an Isolated Management Plane: A Zero-Trust Redesign of Home Network Administration

## Why This Project Exists

Every host in this lab used to be reachable from my daily-driver PC. That's a
normal, unremarkable way to run a homelab — and it's also, structurally, the
same mistake that shows up in breach postmortems at real companies: the
workstation someone browses the web on is the same workstation that can SSH
into production.

The question that started this project wasn't "how do I add more services."
It was: **if my daily driver gets compromised — a bad browser extension, a
phishing link, a supply-chain hit on some tool I installed — what can an
attacker reach?**

The honest answer, before this work, was: everything. Every hypervisor,
every backup target, the router itself. That's not a homelab problem, it's
an architecture problem, and it's the same one enterprise security teams
spend real budget solving with jump hosts, bastions, and privileged access
management. I wanted to solve it the same way — not because a homelab
needs enterprise rigor, but because building it that way is the entire
point of using this as a portfolio piece.

This document is about the *design decisions* — what I chose, what I
rejected, and why — with enough of the actual mechanics included that a
technical reader can see how each decision was implemented, not just
that it was made.

---

## The Core Design Principle

**No general-purpose device should ever hold standing credentials to
management infrastructure.**

That single sentence drove almost every decision in this project. It meant:

- The daily-driver PC should not be able to reach hypervisors, backup
  servers, or the router directly.
- The one thing that *can* reach everything (a jump host) should do
  nothing else — no browsing, no email, no general use, so its attack
  surface stays small and auditable.
- Authentication should be hardware-backed (FIDO2) wherever a human is
  involved, and certificate-based wherever a machine is involved — never a
  password sitting in a config file.
- Every credential should be scoped to the minimum it needs, and every
  script/service account should be able to do *less* than the human who
  set it up, not the same amount.

Everything below is that principle, applied to a real, messy, evolving
network — with the specific mechanisms that made it real.

---

## Architecture: The Shape of the Solution

### A dedicated management VLAN

Management traffic (hypervisor consoles, backup infrastructure, the
switch and access point admin interfaces, the router itself) lives on its
own VLAN, isolated from every other subnet by the router's firewall — the
*only* enforcement layer in this design. There are no per-VM or
per-container firewalls anywhere in the network.

That's a deliberate rejection of a common "more layers is always better"
instinct. Docker's networking model manipulates the host's own iptables
rules directly, in ways that can silently bypass a host-level firewall
(ufw, firewalld) configured alongside it — a container can end up exposed
on a port the host-level firewall thinks it's blocking. Stacking a
host firewall on top of that doesn't add real defense; it adds a rule set
that *looks* authoritative but isn't, which is worse than no host firewall
at all, because it invites false confidence during an audit. The router's
firewall, sitting outside the host entirely, doesn't have this blind spot
— it sees actual layer-3/4 traffic regardless of what any given host's
container runtime is doing internally. One real enforcement point beats
two enforcement points where one of them lies to you.

Practically, this means every new VLAN added to the network requires an
explicit allow rule before any traffic crosses it — the switch trunks
everything by default, so segmentation is enforced entirely at the
router, and a forgotten rule fails closed rather than open.

### A bastion host — and *only* a bastion host

A dedicated jump server sits at the front of the management VLAN, running
exactly two functions: hosting the SSH certificate authority, and
forwarding authenticated sessions onward. No other service runs on it.

**The CA mechanism itself, concretely:** the fleet uses OpenSSH's
certificate authority feature — a pair of CA keypairs (one for signing
user identities, one for signing host identities), rather than the more
common model of distributing individual public keys to every
`authorized_keys` file. Every fleet host is configured with
`TrustedUserCAKeys` pointing at the user CA's public key, and an
`AuthorizedPrincipalsFile` mapping a certificate's embedded principal
name to a local login. This means adding a new trusted identity, or
revoking one, never requires touching every host's `authorized_keys`
individually — trust is established once, structurally, and every
signed certificate is honored fleet-wide the moment it's presented.

Host identity works the same way in reverse: every host's SSH daemon
presents a certificate (not a bare key) signed by a host CA, and every
client trusts that CA via a single `@cert-authority` line in
`known_hosts` — eliminating the usual "unknown host key, are you sure?"
trust-on-first-use prompt without weakening verification, since the
certificate is cryptographically tied to the CA rather than blindly
accepted on first contact.

**The bastion doesn't trust its own CA.** Every other host in the fleet
trusts certificates signed by this CA — the bastion's own `sshd_config`
deliberately omits `TrustedUserCAKeys`/`AuthorizedPrincipalsFile`,
so inbound access to the bastion itself relies on a hardware-backed
FIDO2 key (`authorized_keys`, not certificate trust) instead. The
reasoning: the machine that *issues* trust shouldn't also blindly consume
it. If the CA signing key were ever compromised, forged certificates
still couldn't get an attacker back into the box holding the CA itself
— that door uses a different lock entirely.

**The CA private key never touches persistent disk.** Signing a
certificate follows a fixed, repeatable sequence: mount a `tmpfs` RAM-
backed filesystem on the bastion, decrypt the CA private key from an
offline, `age`-encrypted USB drive directly into that `tmpfs`, run
`ssh-keygen -s` to sign the target public key, copy the resulting
certificate out to its destination, then `shred -u` the decrypted key and
unmount the `tmpfs`. At no point does an unencrypted copy of the signing
key exist anywhere but volatile memory, and the whole sequence requires
physically connecting the offline key drive — no daemon, cron job, or
remote trigger can invoke it.

### An isolated browser appliance

Every management interface in this network has a web GUI. Somewhere has
to run a browser to reach them, and putting one on the bastion would have
undone the entire point of keeping it narrow — browsers are enormous,
frequently-patched attack surfaces, and pairing one with SSH signing
trust on the same host would be exactly the anti-pattern this project set
out to eliminate.

Instead, a second, purpose-built appliance VM handles this: a minimal
Debian install, one desktop environment, one browser, no general web
access or extensions beyond a password manager. It's reached via a
**nested SSH tunnel**, not direct exposure: the client authenticates
once, with a hardware key, to the bastion; the bastion then opens a
second connection to the appliance *on its own certificate-based
identity*, not the client's — meaning a single hardware-key touch grants
access all the way through, without the client ever needing standing
credentials to the appliance itself. (An earlier version of this used
SSH's `ProxyJump`, which *looks* similar but authenticates the client's
own identity twice, once per hop — a subtle but meaningful difference:
`ProxyJump` means the client's credential is what's trusted at the final
hop, while a nested command means the bastion is vouching for the client,
and the final hop's trust never leaves the bastion.)

If this appliance is ever compromised — the most exposed host in the
design, by nature of running a browser — the blast radius is contained to
a locked-down VM with no stored credentials of its own to anything else
in the fleet, reachable by nothing beyond its own tunnel.

### A dedicated automation identity — deliberately less trusted than me

Routine fleet operations run under a dedicated service account, fully
separate from my personal identity, authenticating fleet-wide via its
own CA-signed certificate (a second certificate issued off the same user
CA, bound to a different principal name). Its `sudo` rights are scoped
per command, not per host — e.g. a specific `apt-get purge` invocation
pattern, nothing broader — enforced via `/etc/sudoers.d/` drop-in files
rather than a blanket grant. The rule followed throughout: grant exactly
what today's task needs, expand later when a new task genuinely requires
it, never provision ahead of an actual need.

That narrow-by-default philosophy is what eventually collided with a real
technical limitation — covered below.

### An out-of-band physical fallback

Every layer above depends on the network being healthy enough to reach
the bastion. The fallback for when it isn't: a specific, normally-
administratively-disabled switch port, configured with the management
VLAN as its native (untagged) VLAN and explicitly excluded from the
switch's trunk/LAG membership. Because this is a pure Layer 2 access
port, reaching it requires no routing, no DHCP, and no dependency on the
router functioning at all — a physical cable move and a manually-assigned
static address are enough to land directly on the management network's
broadcast domain. It's enabled only for the duration of active use and
disabled again immediately after, and it's paired with two other
independent recovery paths (hypervisor console access, which sidesteps
guest networking entirely; and direct physical access to the router
appliance) — each covering a distinct failure mode the others don't.

---

## Where the Design Got Tested — and What Broke

A design only proves itself when it collides with something real. Three
moments in this project were worth documenting specifically because they
represent the difference between *intending* a security property and
*verifying* one.

### The automation trust boundary, tested to its actual limit

The plan called for automating one more task: annual certificate renewal
for every host in the fleet. In principle this should have extended the
existing service-account automation cleanly — copy a signed certificate
into place, restart `sshd`, same shape as the already-working, narrowly-
scoped tasks.

It didn't. Ansible's `become` privilege-escalation layer — regardless of
which module executes the actual command (`command`, `copy`, `raw`, even
a dedicated wrapper script referenced by a fixed path) — always
constructs its actual remote invocation as something like:

```
sudo -H -S -n -u root /bin/sh -c 'echo BECOME-SUCCESS-<random-marker>; <command>'
```

That `echo BECOME-SUCCESS-<marker>` prefix is how Ansible detects whether
escalation succeeded — it's baked into the framework's own internals, not
something a playbook can opt out of. The practical consequence: `sudo` is
never being asked to authorize the target command directly. It's always
authorizing `/bin/sh -c '...'`, with a randomly-generated marker baked
into the string on every single run. No sudoers rule — however precisely
the target command is written, whether pattern-matched with a wildcard or
scoped to one fixed wrapper script's path — can match a string that
changes its content every invocation. This was confirmed directly:
`sudo -l -U ansible` on a target host showed the exact, correctly-scoped
grant already in place; the failure persisted regardless.

Two ways through this were real options. Grant the automation account
`NOPASSWD: ALL` (or the functionally-equivalent `/bin/sh -c *`) — trivial,
immediate fix, and a direct reversal of the least-privilege principle the
whole account exists to enforce. Or use SSH's `command=` forced-command
option in `authorized_keys` to give the automation account a *direct
root login*, restricted at the SSH layer (not sudo) to executing exactly
one fixed script — technically sound, genuinely narrow, and still
rejected, because it introduces a standing root-capable credential
outside the certificate trust model entirely, solving one problem by
quietly working around the system built to prevent exactly this kind of
credential sprawl.

The actual resolution: certificate renewal doesn't run through the
automation account at all. It's a plain shell script, invoked
interactively, using my own already-privileged identity — because that's
what the task genuinely is: a deliberate, once-a-year, human-present
event that already requires physically unlocking the offline CA key.
Trying to force it into the unattended-automation model wasn't fixing a
bug; it was applying the wrong trust model to a task that never needed
one in the first place. Recognizing that distinction, after real
investigation rather than as a shortcut past a hard error, was the
actual outcome worth keeping.

### An inconsistency that would have undermined its own goal

Hardware configuration backups — network device configs containing
things like local admin credentials and pre-shared keys — were stored in
private, access-controlled Git repositories: reasonable, but not
encrypted at rest. The natural next step: add transparent,
repository-level encryption (`git-crypt`, which wires an encrypt/decrypt
filter into Git's own `clean`/`smudge` mechanism via `.gitattributes`, so
`git add`/`commit` encrypts automatically and the repository host itself
never sees plaintext).

That setup hit — and surfaced — a genuinely subtle bug worth documenting
on its own: a `.gitattributes` rule of `* filter=git-crypt` matches every
file in the repository, including `.gitattributes` itself. Once that file
is encrypted, Git can no longer read the very rule instructing it to
apply the filter — a self-referential lock-out that silently disabled
encryption for every other file in the repo too, while still *looking*
configured correctly (the commands all exited cleanly, no errors). It
was only caught by verifying the actual stored bytes in a fresh clone,
not by trusting a successful exit code — confirming genuine ciphertext
was present before testing decryption, not just after. The fix is a
second, explicit exclusion line:

```
* filter=git-crypt diff=git-crypt
.gitattributes !filter !diff
```

With that fixed and independently re-verified (ciphertext before unlock,
correct plaintext after, using the actual offline backup copy of the
decryption key — not just the working copy still sitting on the machine
that generated it), the encryption layer was genuinely, provably working.

It was reverted anyway. One device's configuration backup is pushed
automatically by that device's own firmware, using its own internal Git
client running on a different OS entirely — a client with no way to
participate in a locally-configured Git filter, because the filter only
exists in `.git/config` on whichever machine runs the commit, and that
device's push never touches a machine with `git-crypt` installed at all.
Encrypting every manually-managed config while that one automated push
kept landing in plaintext, in the same repository, would have produced
something worse than the starting point: a repository that *looked*
uniformly protected but wasn't — a more dangerous state than one that's
honestly, consistently unencrypted, because it invites trust that isn't
earned. The encrypted approach was objectively stronger in isolation, and
wrong for this specific repository, because it couldn't be applied
uniformly across every writer touching it.

### A gap that had nothing to do with any of this — found anyway

While chasing an unrelated SSH authentication oddity, one host was found
still permitting password-based login — `PasswordAuthentication` absent
from `sshd_config` entirely, meaning it was silently sitting on OpenSSH's
compiled-in default of `yes`, while every other host in the fleet had it
explicitly set to `no`. That host was a pre-built vendor appliance image
rather than a machine provisioned through the standard hardening
playbook, so it had simply never gone through the pass that sets it.

Nothing in this project was looking for that gap specifically — it
surfaced because a connection attempt that should have failed cleanly
(no valid key offered) instead silently fell back to a password prompt
and succeeded. That unexpected success, rather than being accepted as
"it worked," got traced to its actual cause. It's arguably the most
transferable lesson in the whole project: **a security control that
degrades gracefully into "it still worked, somehow" deserves more
suspicion than one that fails loudly.** Loud failures are self-reporting.
Quiet, permissive fallbacks are the ones that sit unnoticed for months.

---

## What This Actually Achieves

Measured against the original question — *what can an attacker reach if
my daily driver is compromised* — the answer is now genuinely different:

- **Nothing, directly.** The daily driver has no route to management
  infrastructure at all — not a firewall rule *permitting* it that could
  theoretically be bypassed, but literal network-layer absence of path.
- **Reaching the bastion requires a hardware security key.** A fully
  compromised OS can't extract or replay a FIDO2-backed credential the
  way it could a stored private key or password.
- **The bastion itself has nothing further to steal.** Downstream access
  is authenticated per-session via short-lived certificate trust, not a
  stored key an attacker could copy off and reuse elsewhere.
- **The CA key that makes all of this trust possible is physically
  offline**, unlocked only via a deliberate, in-person, RAM-only
  decryption ritual.
- **The one component with meaningful attack surface (a browser) is
  isolated on hardware with no credentials of its own to anything else**
  — compromising it yields a locked room, not a master key.
- **The automation account can do measurably less than I can, on
  purpose** — verified directly against its actual sudoers grant, not
  assumed — even where that made one task slower to build than it needed
  to be.

None of this is exotic. It's the same shape every serious security team
uses: least privilege, defense in depth, hardware-backed identity, an
offline root of trust, and a hard boundary between "things that browse
the internet" and "things that hold keys to production." What this
project demonstrates isn't a novel technique — it's the discipline of
applying well-known principles consistently, verifying them at the
protocol/config level rather than trusting a green checkmark, and being
willing to back out a real, working improvement (the encryption layer)
when it stopped agreeing with the principle it was meant to serve.

---

## What I'd Do Differently

Documenting mistakes is only useful if it produces a rule for next time:

- **Set static addressing before any provisioning work begins**, not
  after. Doing it late on one host meant re-signing its host certificate
  and re-editing every SSH config referencing its old address — entirely
  avoidable by sequencing one step earlier, and now the first documented
  step in the standard new-host deployment process.
- **Verify a security property at the artifact level, not by trusting
  exit codes.** A script reporting success, or a config file appearing
  correct at a glance, isn't the same as confirming the actual bytes on
  disk or the actual string a permission system is evaluating — more
  than one issue in this project hid behind a clean exit status until
  checked directly.
- **Decide a task's trust model from its actual nature, not its
  tooling category.** "This is Ansible work" was the wrong lens for
  certificate renewal; "this is a once-a-year, human-present event" was
  the right one, and using the wrong lens cost real, avoidable
  troubleshooting time before the reframe.
- **A rejected security improvement is still a successful outcome**, as
  long as the rejection is reasoned rather than a shortcut past
  friction. The `git-crypt` reversal wasn't a failure to document
  quietly — it's one of the more useful decisions in this write-up,
  precisely because the alternative (keeping it, inconsistency and all)
  would have looked like progress while actually being worse.
