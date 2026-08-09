# air-gap-the-keys
A zero-trust management network: SSH certificate authority, hardware-backed auth, isolated jump host, and an offline root of trust — built for a home lab, designed like production infrastructure.

A from-scratch redesign of how a home lab's administrative surface is
reached — built around one question: *if my daily-driver PC is
compromised, what can an attacker reach?* Before this project, the
answer was "everything." This repository documents the redesign that
made the answer "nothing, directly."

The approach borrows directly from how real security teams protect
production access: a hardened jump host, certificate-based trust instead
of long-lived keys, hardware-backed authentication, an offline root of
trust, and strict least-privilege for anything that runs unattended.
None of the individual techniques here are novel — the value is in
applying them consistently, verifying them at the protocol level instead
of trusting a green checkmark, and being honest about the places a good
idea got rejected once it collided with reality.

## Start Here

**[mgmt-vlan-isolation-project.md](./mgmt-vlan-isolation-project.md)**
The main write-up. Covers the architecture end to end — the management
VLAN, the bastion and its certificate authority, the isolated browser
appliance, the deliberately-underprivileged automation account, and the
physical break-glass fallback. Includes a retrospective on three points
where the design was actually tested: an automation framework that
turned out to be structurally incompatible with a narrow trust boundary,
a security improvement that got built, verified, and then reverted on
principle, and a hardening gap that surfaced by accident while chasing
something unrelated. Read this first — everything else in this
repository is a deeper dive on one piece of it.

## Walkthroughs

**[walkthroughs/ssh-ca-setup.md](./walkthroughs/ssh-ca-setup.md)**
How to build the SSH certificate authority that the rest of this
architecture depends on — both a user CA and a host CA, the offline
signing ritual, and why the bastion is deliberately excluded from
trusting its own CA.

**[walkthroughs/bastion-tunnel-single-tap-auth.md](./walkthroughs/bastion-tunnel-single-tap-auth.md)**
How a single hardware-key touch grants access through two SSH hops —
the technique behind reaching the isolated browser appliance for RDP,
and more broadly, a reusable pattern for "the jump host vouches for you"
access versus "you authenticate twice" access. Includes the actual
launcher script and the mistakes made building it.

## Notes

**[notes/git-crypt-decision.md](./notes/git-crypt-decision.md)**
Repository-level encryption for hardware config backups was built,
verified working end to end, and then deliberately reverted. Short
note on why — not a walkthrough, since the outcome was "don't do this
here," but the reasoning behind that call is worth keeping.

## License

MIT. Everything here is sanitized — no real IP addresses, hostnames,
domains, or vendor-identifying details. Adapt the patterns; the
specifics are placeholders.
