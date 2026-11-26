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
that it was made. It describes the design as it runs today, including the
parts that changed after the first version met reality.

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
- No automation holds administrative reach into the fleet beyond
  looking. Changes happen when I'm present, under my own identity,
  after a dry run.

Everything below is that principle, applied to a real, messy, evolving
network — with the specific mechanisms that made it real.

---

## Architecture: The Shape of the Solution

### A dedicated management VLAN

Management traffic (hypervisor consoles, backup infrastructure, the
switch and access point admin interfaces, the router itself) lives on its
own VLAN. The router's firewall is the primary boundary: every other
subnet reaches the management VLAN only through explicit allow rules.
The switch trunks everything by default, so segmentation is enforced
entirely at the router and a forgotten rule fails closed rather than
open. The rules for my own machines are short and named: from the daily
drivers, the only paths into the management VLAN are SSH to the bastion,
HTTPS to the management server's web front doors, and the monitoring
agents' reports to the log collector.

Behind the router sit two further layers, placed deliberately where they
earn their keep rather than everywhere:

- **Each hypervisor runs its own firewall**, rendered from a single
  template that is identical on every node and checked by a playbook. It
  allows only what the template lists plus the hypervisor's built-in rule
  for its own management network, which means a mistake in the template
  can't lock the management network out. This layer was off for a while:
  an early, inconsistently-applied version was missing an allow rule for
  the management server on the nodes where it was enabled, and the
  resulting SSH timeouts looked exactly like a network fault. I switched
  it off to get back to a known state, then brought it back as deliberate
  defense in depth once it could be produced from one template and
  verified, not hand-built per node.
- **The one internet-facing host runs a host firewall.** An earlier
  version of this document argued against host firewalls altogether,
  because Docker manipulates the host's own iptables rules and can
  bypass a firewall configured alongside it. That was half right.
  Published container ports really do skip a host firewall's input
  rules, since they are forwarded rather than delivered to the host. The
  right conclusion was to write the rule where Docker actually looks
  (the `DOCKER-USER` chain), not to have no host firewall. The one
  published management-agent port is restricted there to a single
  source, and the rule was verified to survive a reboot, because a second
  layer that has never been tested is only a rule that looks
  authoritative.

The bastion and the management server have no host firewall. For them,
segmentation at the router is the chosen control: it decides what reaches
them and where they can go.

Every fleet host also refuses SSH forwarding of any kind (`DisableForwarding`,
set fleet-wide). Nothing in the fleet uses jump-host chaining, agent
forwarding or port forwards; I SSH onward from the bastion's own shell.
That removes a whole class of pivot paths, and it is the reason an
earlier tunnel technique in this repository is now archived.

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
name to a local login. Adding a new trusted identity, or retiring one,
never requires touching every host's `authorized_keys` individually —
trust is established once, structurally, and every signed certificate is
honored fleet-wide the moment it's presented.

Host identity works the same way in reverse: every host's SSH daemon
presents a certificate (not a bare key) signed by the host CA, valid for
its short name, its full name and its IP address. The fleet's own SSH
clients trust that CA through a single `@cert-authority` line in
`known_hosts`, which removes the "unknown host key, are you sure?"
prompt without weakening verification.

**Where the CA deliberately doesn't reach.** A few things sit outside
the certificate system, each for a concrete reason:

- The NAS only permits root logins, and its `AllowUsers` rule rejects a
  certificate principal before principals are even consulted, so it is
  reached with a dedicated raw key.
- The router's SSH configuration can be rewritten by a firmware or
  package update, so it keeps an ordinary key and manual host trust
  rather than certificate files that might silently disappear.
- My workstations pin the bastion's fingerprint instead of trusting the
  host CA.

Documenting the exceptions matters as much as the rule: an exception
nobody wrote down is how a gap hides.

**The bastion doesn't trust its own CA.** Every other host in the fleet
trusts certificates signed by this CA — the bastion's own `sshd_config`
deliberately omits `TrustedUserCAKeys`/`AuthorizedPrincipalsFile`, so
inbound access relies on hardware-backed FIDO2 keys (`authorized_keys`,
not certificate trust): a primary and a backup. The one other inbound
path is declared rather than hidden: the automation account's raw key
from the management server, which can run two read-only audit commands.
The reasoning: the machine that *issues* trust shouldn't also blindly
consume it. If the CA signing key were ever compromised, forged
certificates still couldn't get an attacker back into the box holding
the CA itself — that door uses a different lock entirely.

**The bastion's own outbound identity** is a certificate with no expiry.
That's a trade-off, not a detail: it can't be retired by waiting for it to lapse. What bounds it is
where it lives (only on the bastion, which accepts only hardware keys
inbound) and what it grants: a login as my own account, with anything
privileged still requiring my sudo password, typed per run.

**The CA private key never touches persistent disk.** Signing follows a
fixed, repeatable sequence on the bastion: a RAM-backed `tmpfs` owned by
the signing user, the `age`-encrypted CA key copied in from an offline
drive and decrypted straight into it, its fingerprint checked against a
pinned value *before* anything is signed, then signing, then `shred -u`
and an unmount that is verified rather than assumed. The bastion runs
with no swap and with hibernation resume disabled and memory ballooning
off, so "RAM-only" is literally true. The whole sequence requires
physically connecting the offline drive and typing a passphrase — no
daemon, cron job, or remote trigger can invoke it.

### A browser for management web interfaces

Every management interface in this network has a web GUI. Somewhere has
to run a browser to reach them, and putting one on the bastion would have
undone the entire point of keeping it narrow.

The first design was a purpose-built appliance VM, reached through a
nested SSH tunnel and RDP. It was easy to lock down and awkward to live
with: it needed more resources than it deserved, keeping it isolated to
local-only meant constantly switching windows (read a value in one,
minimize, edit in another, maximize), and it meant a second password
manager. I replaced it with a hardened browser container on the
management server, which is simpler to run and has one password manager
instead of two.

That change traded away the hypervisor boundary, so the old claim —
"a locked room with no credentials of its own" — had to be re-earned
rather than carried over. How that went is the last item in the
retrospective below. The result:

- It sits behind two authentication layers: single sign-on first (set up
  in [one-front-door](https://github.com/zacharyrichter/one-front-door)), then the browser's own local login.
- It has its own network. Only the reverse proxy shares it; no other
  container can reach it, and it can't reach theirs.
- It mounts only its own profile and downloads folders: no Docker socket,
  no access to the host's keys or repositories.
- The reverse proxy only serves it to four source addresses: the two
  administrative machines on their normal networks, the laptop's VPN
  address, and the desktop's out-of-band address. A phone that may use
  the other management web interfaces gets nothing here.
- A small helper I wrote opens bookmarks as browser tabs, restricted to
  an allow-list of destinations.

It is still the most exposed component in the design, and a browser
session holding the password manager is the most valuable thing on its
host. The controls above are what stand between a compromised browser
and everything else; I don't claim separate hardware.

### A dedicated automation identity — deliberately less trusted than me

Routine fleet operations run under a dedicated service account, fully
separate from my personal identity, authenticating fleet-wide via its
own CA-signed certificate (a second certificate issued off the same user
CA, bound to a different principal name). It is for **auditing only**:
its `sudo` rights are exact, read-only commands with no wildcards,
enforced via `/etc/sudoers.d/` drop-in files. Anything that changes a
host — package purges, kernel cleanup, hardening, certificate renewal —
runs as me, with my sudo password, after a dry-run (`--check`) pass I've
read first.

That wasn't always the arrangement. An earlier version gave the account a
narrow purge rule so the cleanup playbooks could run unattended. I
removed it when I decided the account should audit only; package purges
and the cleanup playbooks now run as me, with a dry run first. The only
scheduled job that logs in to other hosts as the automation account is
read-only: a monthly report on certificate expiry.

### An out-of-band physical fallback

Every layer above depends on the network being healthy enough to reach
the bastion. The fallback for when it isn't: a specific, normally-
administratively-disabled switch port, configured with the management
VLAN as its native (untagged) VLAN and explicitly excluded from the
switch's trunk/LAG membership. Because this is a pure Layer 2 access
port, reaching it requires no routing, no DHCP, and no dependency on the
router functioning at all — a physical cable move and a manually-assigned
static address are enough to land directly on the management network's
broadcast domain.

My desktop lives on the user network and joins the management VLAN only
through this port. While it's patched in, it reaches management hosts
directly and holds open access to every administrative network through a
single router rule — which is exactly why the port is disabled the rest
of the time and enabled only for the duration of active use. It's paired
with two other independent recovery paths (hypervisor console access,
which sidesteps guest networking entirely; and direct physical access to
the router appliance) — each covering a distinct failure mode the others
don't.

---

## Where the Design Got Tested — and What Broke

A design only proves itself when it collides with something real. Four
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
`sudo -l -U <account>` on a target host showed the exact, correctly-scoped
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

The resolution came in two steps, and the second matters more than the
first. The first was a plain shell script, run interactively under my own
identity, because certificate renewal is a deliberate, once-a-year,
human-present event that already requires physically unlocking the
offline CA key. That worked, and it also showed what a hand-run script
is bad at: it once reported success while installing a stale certificate
left in a temporary directory by an earlier run, it hung on an
interactive prompt, and its host list lived in the script instead of in
the inventory the rest of the fleet uses.

The second step was to stop treating "human-present" as "not Ansible."
The `become` problem is specific to the service account's narrow sudo
rules. Run the same playbook under my own identity with my own sudo and
`become` works exactly as designed — no wildcard, no forced-command key,
no standing credential. The mistake had never been the tool; it had been
assigning the task to the wrong identity. So renewal is now an Ansible
run on the bastion, using my own sudo, with the safeguards a hand-run
script never had:

- **One source of truth for the host list:** an inventory group, shared
  with the monthly expiry check, so adding a host needs no change to the
  tool.
- **A preflight that never touches the key:** every host is checked for
  reachability, current certificate and principals before anything is
  decrypted. The run also refuses to start if swap is enabled, if a
  previous run left its RAM directory behind, or if the installed tool's
  ownership or permissions are not what they should be.
- **A supply chain that is reviewed:** the tool's code lives in the
  playbook repository. The bastion fetches it with a read-only,
  single-repository token, shows me a diff, and installs only on a typed
  confirmation into a root-owned, read-only directory. Ansible runs with
  a cleared environment and a locked-down configuration that loads only
  built-in modules, so nothing user-writable is loaded while the CA key
  is decrypted.
- **Per-host rollback and proof:** each host gets its old certificate
  kept as a backup, the new one installed, `sshd` checked and *restarted*
  (never reloaded, which fails on hosts with socket-activated SSH), and
  the run then scans the host to confirm the new certificate is actually
  the one being served. A failure after install restores the old one. One
  host failing never stops the others.
- **A key that can't linger:** a trap shreds the decrypted key on every
  exit path, including an interrupt, and the run ends by verifying the
  RAM directory is gone.

My part is moving the encrypted key to the bastion and decrypting it.
After that the run is unattended from signing to verification. It takes
a once-a-year task that used to be tedious and nerve-wracking to get
exactly right, and makes it routine. The first full run covered every
host in the group with no per-host manual step.

I did consider a different route: running a secrets engine as the CA so
certificates are issued on demand. The only part I wanted was automated
renewal, and automating around the offline key delivers that with no new
service to run, while the signing key still exists decrypted only for
the length of a supervised run. The tool itself, the controls around the
key and the full decision record are in
[expire-on-purpose](https://github.com/zacharyrichter/expire-on-purpose).

### An inconsistency that would have undermined its own goal

Hardware configuration backups — network device configs containing
things like local admin credentials and pre-shared keys — were stored in
private, access-controlled Git repositories: reasonable, but not
encrypted at rest. I added transparent, repository-level encryption
(`git-crypt`), found and fixed a subtle bug that silently disabled it,
verified it end to end — and then reverted it. One device pushes its own
backup using its own internal Git client, which can't take part in a
locally-configured filter, so that backup would have landed in plaintext
next to encrypted siblings. A repository that looks uniformly protected
but isn't is more dangerous than one that's honestly, consistently
unencrypted, because it invites trust that isn't earned. The full story,
including the bug, is in [notes/git-crypt-decision.md](./notes/git-crypt-decision.md).

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

It also stopped being a matter of luck. Later I put the baseline in code:
one shared SSH configuration installed as a drop-in on every host, each
host's main configuration reset to stock, and a playbook that can be
re-run in check mode to show any host that has drifted. The next gap of
this kind shows up in a report instead of a surprise. That setup is its
own repo: [diff-is-the-audit](https://github.com/zacharyrichter/diff-is-the-audit).

### An isolation claim that didn't survive a second read

When I came back to rewrite the browser section of this document, I
found that the isolation I'd assumed it had wasn't there. Moving the
browser from its own VM to a container traded away the hypervisor
boundary, and I'd assumed the container's boundary was a fair
replacement. Reading the stack's configuration said otherwise: every
container on the management server shared one Docker network, so the
browser could reach its neighbours — including the socket proxy the
reverse proxy uses to discover containers. What a discovery proxy can
read about a container includes its environment variables.

The fix was structural. The browser got its own network shared only with
the reverse proxy. The socket proxy got an internal-only network shared
with the reverse proxy and nothing else, so it has no route out either.
The two browser routes also got a source-address restriction, and while
I was in there I found one of them wasn't pinned to the HTTPS entry
point and was answering on a second port, which is now closed. Then I
tested the property instead of reading it: from inside the browser
container, the socket proxy's name no longer resolves, and the browser
opens normally from the two administrative machines, including over the
VPN.

I considered mutual TLS on the browser's route instead of a
source-address restriction. It works — TLS options can be scoped to a
single hostname without touching the other management interfaces, which
have to stay reachable from a phone — but it needs a second CA, and its
client key would live on the very machine I'm defending against, plus
per-install client setup on every reinstall. A server-side restriction
gives the same narrowing with nothing to remember.

---

## What This Actually Achieves

Measured against the original question — *what can an attacker reach if
my daily driver is compromised* — the answer is now genuinely different:

- **No path to the hypervisors, the router, or the backup
  infrastructure.** What remains is deliberately small: SSH to the
  bastion, HTTPS to a short list of web front doors on the management
  server, and the monitoring agents' reports to the log collector.
  Everything else is denied at the router, which sits outside any host
  a compromised daily driver could reach.
- **Reaching the bastion requires a hardware security key.** A fully
  compromised OS can't extract or replay a FIDO2-backed credential the
  way it could a stored private key or password.
- **The bastion itself has nothing further to steal.** Downstream access
  is authenticated through certificate trust, not a stored key an
  attacker could copy and reuse elsewhere, and anything privileged still
  needs a password I type.
- **The CA key that makes all of this trust possible is physically
  offline**, unlocked only via a deliberate, in-person, RAM-only
  decryption ritual.
- **The one component with meaningful attack surface (a browser) is
  contained by controls, not by hardware** — two authentication layers,
  its own network, no access to host secrets, a source-address
  restriction — and I've tested the isolation instead of asserting it.
- **The automation account can do measurably less than I can, on
  purpose** — read-only, verified directly against its actual sudoers
  grant, not assumed — even where that made one task slower to build than
  it needed to be.
- **The state described here is checked, not remembered.** Standards are
  playbooks, and audits run in check mode report drift
  ([diff-is-the-audit](https://github.com/zacharyrichter/diff-is-the-audit)).

None of this is exotic. It's the same shape every serious security team
uses: least privilege, defense in depth, hardware-backed identity, an
offline root of trust, and a hard boundary between "things that browse
the internet" and "things that hold keys to production." What this
project demonstrates isn't a novel technique — it's the discipline of
applying well-known principles consistently, verifying them at the
protocol/config level rather than trusting a green checkmark, and being
willing to back out a real, working improvement (the encryption layer),
or to retract a claim of my own (the browser's isolation), when it
stopped agreeing with the principle it was meant to serve.

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
  disk, the actual string a permission system is evaluating, or the
  actual network a container sits on — more than one issue in this
  project hid behind a clean exit status until checked directly.
- **Decide a task's trust model from its actual nature, not its
  tooling category.** "This is Ansible work" was the wrong lens for
  certificate renewal; "this is a once-a-year, human-present event" was
  the right one. The tooling question only became easy after that, and
  it ended with Ansible anyway, under the right identity.
- **Test an isolation claim, don't draw it.** A diagram with a box
  around the browser is not a network boundary. Two lines of output from
  inside the container were.
- **A rejected security improvement is still a successful outcome**, as
  long as the rejection is reasoned rather than a shortcut past
  friction. The `git-crypt` reversal wasn't a failure to document
  quietly — it's one of the more useful decisions in this write-up,
  precisely because the alternative (keeping it, inconsistency and all)
  would have looked like progress while actually being worse.
