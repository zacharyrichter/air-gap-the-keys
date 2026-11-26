# Walkthrough: Building an SSH Certificate Authority

This is the mechanism the rest of the isolated management plane depends
on. Instead of distributing individual public keys to every host's
`authorized_keys` (and every client's `known_hosts`), a small number of
signing keys establish trust structurally — add a host to the fleet, and
it's trusted the moment its certificate is signed, with no per-host key
distribution step.

Two separate certificate authorities are used: one signs **user**
identities (so a person or service account can log in), one signs
**host** identities (so a client can verify it's really talking to the
server it thinks it is, without a manual "trust this fingerprint?"
prompt). They're independent — compromising one doesn't compromise the
other.

## Why not just distribute public keys the normal way?

Public-key-only SSH works fine at small scale. It stops scaling cleanly
once you have more than a handful of hosts and more than one identity
that needs access to some-but-not-all of them: every new host means
touching every relevant `authorized_keys` file, and revoking access
means finding and removing a key from every file it was ever added to.
A CA collapses that into "sign a certificate" and "retire by expiry or
by removing trust in one place" — the same reason real infrastructure
uses CAs instead of manually distributed keys.

## Generating the CA keypairs

Done once, on an offline or otherwise trusted machine — these keys are
sensitive enough that they should never live on a routinely-connected
host:

```bash
ssh-keygen -t ed25519 -f example-user-ca -C "user CA"
ssh-keygen -t ed25519 -f example-host-ca -C "host CA"
```

This produces four files: two private keys, two public keys. The
private keys are the actual root of trust — treat them the way you'd
treat a root CA certificate's private key in any PKI, because that's
exactly what they are. Record each CA's fingerprint
(`ssh-keygen -lf example-user-ca.pub`) somewhere separate from the
encrypted private key; you'll check against it every time the key is
decrypted.

## Setting up trust on the fleet

**User CA — every server that should accept certificate-based logins.**
Put the settings in a drop-in, not the main `sshd_config`, so every host
carries the same small file and the main configuration can stay stock:

```
# /etc/ssh/sshd_config.d/10-fleet.conf
TrustedUserCAKeys /etc/ssh/ca/example-user-ca.pub
AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u
```

`TrustedUserCAKeys` tells `sshd` which CA's signature it will accept on
an incoming certificate. `AuthorizedPrincipalsFile` maps a certificate's
embedded *principal* (an identity name baked into the cert at signing
time — not necessarily the same as the Unix username) to the local
account being logged into. Each file, e.g.
`/etc/ssh/auth_principals/admin`, just contains the principal name(s)
that account should accept:

```
admin
```

One thing to check if a certificate login is refused before principals
are even consulted: an `AllowUsers` rule in `sshd_config` rejects a
login by account name first. Any account you want certificates to work
for has to be listed there too.

**Host CA — every client that should trust host certificates without
prompting:**

```
# ~/.ssh/known_hosts
@cert-authority * <contents of example-host-ca.pub>
```

The `*` scopes this to any hostname — sshd's own certificate (signed by
this CA) is what actually asserts *which* host it is; the client just
needs to trust the CA once, not maintain a growing list of individual
host fingerprints. This line has to be added to every SSH client that
should use it, explicitly; it does not propagate on its own, and without
it `ssh` silently falls back to trust-on-first-use prompts.

## Signing a host certificate

Every host that will serve SSH gets its own certificate, binding its
host key to every name it will be reached by, and an expiry:

```bash
ssh-keygen -s example-host-ca \
  -I "some-hostname.example.org" \
  -h \
  -n "some-hostname,some-hostname.example.org,10.0.99.5" \
  -V -5m:+52w \
  -z 20261103120001 \
  /etc/ssh/ssh_host_ed25519_key.pub
```

- `-h` marks this as a **host** certificate (as opposed to a user cert).
- `-n` lists every name the certificate should be valid for. Include the
  **IP address**: `ssh` validates the certificate against the name it
  actually connected to, so if you connect by IP and the IP isn't a
  principal, the certificate is rejected and `ssh` quietly falls back to
  a trust-on-first-use prompt. The flip side is that a host's address
  must be final before you sign, because changing it later means
  re-signing.
- `-V -5m:+52w` makes the window start five minutes in the past, so
  clock drift can't make a fresh certificate look not-yet-valid, and run
  for a year to match an annual renewal cadence.
- `-I` is the key ID; using the full hostname for every host keeps audit
  logs readable. `-z` sets an explicit serial; a timestamp plus the
  host's position in the inventory keeps them unique, and a distinct
  suffix for hand-signed certificates makes them stand out later.

The resulting `<keyname>-cert.pub` goes alongside the host's existing
key, referenced in the same drop-in:

```
HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub
```

After installing a new certificate, **restart** `sshd`, never reload it.
On hosts where SSH is socket-activated, a reload fails with "Cannot bind
any address" and takes the service down until the next connection
restarts it.

## Signing a user certificate

Same shape, without `-h`, and with `-n` specifying the *principal* (the
identity name), not a hostname:

```bash
ssh-keygen -s example-user-ca \
  -I "some-identity-label" \
  -n admin \
  -V +52w \
  some-user-key.pub
```

A certificate can be issued with no expiry at all (`-V forever` is not
a real flag — omit `-V` for a cert with no `valid_after`/`valid_before`
constraint). In this setup the bastion's own outbound identity is
treated that way, a deliberate trade-off: such a certificate can't be
retired by waiting for it to lapse, so what bounds it is where its
private key lives and what the principal is allowed to do. Use it
sparingly, and only where you can say both.

## The part that actually matters: keeping the CA private key offline

None of the above is meaningfully different from any SSH CA tutorial.
The part that makes it a real security boundary, rather than a
convenience feature, is how the private key is handled:

1. The CA private keys live only as `age`-encrypted files on an offline
   USB drive — never on any routinely-connected machine's disk.
2. Signing happens on the bastion, but only ever inside a `tmpfs`
   (RAM-backed, never touches persistent disk), and the bastion has no
   swap and cannot resume from hibernation, so RAM really is RAM-only.
   Do all of it in one sitting.

```bash
# a small RAM-only directory, readable only by you
sudo mkdir -p /mnt/ca-tmp
sudo mount -t tmpfs -o size=1m,mode=700,uid=$(id -u),gid=$(id -g) tmpfs /mnt/ca-tmp
mount | grep ca-tmp          # exactly one tmpfs line

# from the machine with the offline drive: copy ONLY the CA you need
scp /path/to/drive/keys/example-host-ca.age bastion:/mnt/ca-tmp/

# on the bastion: decrypt, then verify before doing anything else
cd /mnt/ca-tmp
age --decrypt -o host_ca example-host-ca.age
chmod 600 host_ca
ssh-keygen -lf host_ca       # must match the fingerprint you recorded
```

If the fingerprint is not the CA you meant to use, you decrypted the
wrong one: do not sign, go straight to the cleanup. Never copy both CAs
in the same session.

```bash
# sign whatever needs signing (one line per command)
ssh-keygen -s /mnt/ca-tmp/host_ca -I "..." -h -n "..." -V -5m:+52w some-host.pub

# clean up, even if something went wrong
cd ~
shred -u /mnt/ca-tmp/host_ca /mnt/ca-tmp/example-host-ca.age
sudo umount /mnt/ca-tmp
ls /mnt/ca-tmp; mount | grep ca-tmp      # both should print nothing
```

`shred -u` overwrites the file before deleting it, and unmounting the
`tmpfs` discards whatever's left in RAM. If your session drops
mid-routine the key stays in RAM until the `tmpfs` is unmounted or the
machine reboots, so reconnect and run the cleanup. At no point does the
decrypted private key exist anywhere but volatile memory, for the
duration of one signing session, on one machine, requiring the offline
drive to be physically present.

## The bastion doesn't trust its own CA

This is a small but important asymmetry: the bastion — the machine that
*holds and uses* the CA — deliberately does **not** configure
`TrustedUserCAKeys`/`AuthorizedPrincipalsFile` for itself. Inbound
access to the bastion relies on hardware security keys
(`authorized_keys`, ordinary public-key auth) instead of certificate
trust.

The reasoning: if the CA signing key were ever compromised, an attacker
who could forge certificates still shouldn't be able to use one of those
forgeries to get into the box that holds the CA itself. Making the
bastion's own inbound trust independent of the certificate system it
operates closes that loop.

## Renewal

Host certificates expire a year after signing. Renewal does not repeat
the manual ritual above for every host; it is a playbook run on the
bastion that signs, installs and verifies each certificate, so that the
only manual steps are moving the encrypted key onto the bastion and
decrypting it. The manual routine above remains for two cases: signing
a user key, and signing a host certificate if the renewal tool itself is
unavailable. How the renewal run works — and why it is a human-present
run rather than an unattended one — is in the main write-up's
retrospective section, and the tool and its runbook are in
[expire-on-purpose](https://github.com/zacharyrichter/expire-on-purpose).
