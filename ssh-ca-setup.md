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
A CA collapses that into "sign a certificate" and "revoke by expiry or
by removing trust in one place" — the same reason real infrastructure
uses CAs instead of manually distributed keys.

## Generating the CA keypairs

Done once, on an offline or otherwise trusted machine — these keys are
sensitive enough that they should never live on a routinely-connected
host:

```bash
ssh-keygen -t ed25519 -f richterhome-user-ca -C "user CA"
ssh-keygen -t ed25519 -f richterhome-host-ca -C "host CA"
```

This produces four files: two private keys, two public keys. The
private keys are the actual root of trust — treat them the way you'd
treat a root CA certificate's private key in any PKI, because that's
exactly what they are.

## Setting up trust on the fleet

**User CA — every server that should accept certificate-based logins:**

```
# /etc/ssh/sshd_config
TrustedUserCAKeys /etc/ssh/ca/richterhome-user-ca.pub
AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u
```

`TrustedUserCAKeys` tells `sshd` which CA's signature it will accept on
an incoming certificate. `AuthorizedPrincipalsFile` maps a certificate's
embedded *principal* (an identity name baked into the cert at signing
time — not necessarily the same as the Unix username) to the local
account being logged into. The file itself, e.g.
`/etc/ssh/auth_principals/zach`, just contains the principal name(s)
that account should accept:

```
zach
```

**Host CA — every client that should trust host certificates without
prompting:**

```
# ~/.ssh/known_hosts
@cert-authority * <contents of richterhome-host-ca.pub>
```

The `*` scopes this to any hostname — sshd's own certificate (signed by
this CA) is what actually asserts *which* host it is; the client just
needs to trust the CA once, not maintain a growing list of individual
host fingerprints.

## Signing a host certificate

Every host that will serve SSH gets its own certificate, binding its
host key to its hostname(s) and an expiry:

```bash
ssh-keygen -s richterhome-host-ca \
  -I "some-hostname" \
  -h \
  -n "some-hostname,some-hostname.example.org,10.0.0.5" \
  -V +52w \
  /etc/ssh/ssh_host_ed25519_key.pub
```

- `-h` marks this as a **host** certificate (as opposed to a user cert).
- `-n` lists every name/IP the certificate should be valid for — a
  client connecting by any of these will accept it.
- `-V +52w` sets a one-year validity window, matching an annual renewal
  cadence.

The resulting `<keyname>-cert.pub` goes alongside the host's existing
key, referenced in its own `sshd_config`:

```
HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub
```

## Signing a user certificate

Same shape, without `-h`, and with `-n` specifying the *principal* (the
identity name), not a hostname:

```bash
ssh-keygen -s richterhome-user-ca \
  -I "some-identity-label" \
  -n zach \
  -V +52w \
  some-user-key.pub
```

A certificate can be issued with no expiry at all (`-V forever` is not
a real flag — omit `-V` for a cert with no `valid_after`/`valid_before`
constraint) for a long-lived service identity that's otherwise tightly
scoped in what it's allowed to do — used sparingly, and only where the
tradeoff is deliberate.

## The part that actually matters: keeping the CA private key offline

None of the above is meaningfully different from any SSH CA tutorial.
The part that makes it a real security boundary, rather than a
convenience feature, is how the private key is handled:

1. The CA private keys live only as `age`-encrypted files on an offline
   USB drive — never on any routinely-connected machine's disk.
2. Signing happens on the bastion, but only ever inside a `tmpfs`
   (RAM-backed, never touches persistent disk):

```bash
# mount a small RAM-backed filesystem
sudo mount -t tmpfs -o size=10M tmpfs /mnt/ca-tmp
cd /mnt/ca-tmp

# decrypt the CA key from the offline drive, directly into tmpfs
age --decrypt -o richterhome-user-ca richterhome-user-ca.age

# sign whatever needs signing
ssh-keygen -s richterhome-user-ca -I "..." -n "..." -V +52w some-key.pub

# copy the result out, then destroy the decrypted key
cp some-key-cert.pub /wherever/it/needs/to/go/
shred -u richterhome-user-ca

# clean up
cd /
sudo umount /mnt/ca-tmp
```

`shred -u` overwrites the file before deleting it, and unmounting the
`tmpfs` discards whatever's left in RAM. At no point does the decrypted
private key exist anywhere but volatile memory, for the duration of one
signing session, on one machine, requiring the offline drive to be
physically present.

## The bastion doesn't trust its own CA

This is a small but important asymmetry: the bastion — the machine that
*holds and uses* the CA — deliberately does **not** configure
`TrustedUserCAKeys`/`AuthorizedPrincipalsFile` for itself. Inbound
access to the bastion relies on a hardware security key
(`authorized_keys`, ordinary public-key auth) instead of certificate
trust.

The reasoning: if the CA signing key were ever compromised, an attacker
who could forge certificates still shouldn't be able to use one of those
forgeries to get into the box that holds the CA itself. Making the
bastion's own inbound trust independent of the certificate system it
operates closes that loop.

## Renewal

Host certificates expire on the schedule set at signing time (a year, in
this setup). Renewal repeats the same signing ritual for every host,
then distributes and activates each new certificate. That distribution
step — and why it's *not* handled by unattended automation — is its own
story, covered in the main write-up's retrospective section.
