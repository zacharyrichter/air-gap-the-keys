# air-gap-the-keys
A zero-trust management network: SSH certificate authority, hardware-backed auth, isolated jump host, and an offline root of trust — built for a home lab, designed like production infrastructure.

A from-scratch redesign of how a home lab's administrative surface is
reached, built around one question: *if my daily-driver PC is
compromised, what can an attacker reach?* Before this project, the
answer was "everything." Now it is two narrow, authenticated doors — SSH
to one jump host behind a hardware key, and HTTPS to a short list of
management web front doors — plus monitoring agents reporting to a log
collector, and no path at all to the hypervisors, the router, or the
backup infrastructure.

The approach borrows directly from how real security teams protect
production access: a hardened jump host, certificate-based trust instead
of long-lived keys, hardware-backed authentication, an offline root of
trust, and strict least-privilege for anything that runs unattended.
None of the individual techniques here are novel — the value is in
applying them consistently, verifying them at the protocol level instead
of trusting a green checkmark, and being honest about the places a good
idea got rejected, or an isolation claim turned out to be weaker than it
looked, once it collided with reality.

## Start Here

**[mgmt-vlan-isolation-project.md](./mgmt-vlan-isolation-project.md)**
The main write-up. Covers the architecture end to end — the management
VLAN and its layers of enforcement, the bastion and its certificate
authority, the browser used to reach management web interfaces, the
deliberately-underprivileged automation account, and the physical
break-glass fallback. Includes a retrospective on four points where the
design was actually tested: an automation framework that looked
structurally incompatible with a narrow trust boundary until the task
was assigned to the right identity, a security improvement that got
built, verified, and then reverted on principle, a hardening gap that
surfaced by accident while chasing something unrelated, and an
isolation claim that did not survive a second read of its own
configuration. Read this first — everything else in this repository is a
deeper dive on one piece of it.

## Walkthroughs

**[walkthroughs/ssh-ca-setup.md](./walkthroughs/ssh-ca-setup.md)**
How to build the SSH certificate authority that the rest of this
architecture depends on — both a user CA and a host CA, the offline
signing ritual with its RAM-only key handling, and why the bastion is
deliberately excluded from trusting its own CA.

## Notes

**[notes/git-crypt-decision.md](./notes/git-crypt-decision.md)**
Repository-level encryption for hardware config backups was built,
verified working end to end, and then deliberately reverted. Short
note on the bug it surfaced and why the outcome was "don't do this
here" — not a walkthrough, but the reasoning is worth keeping.

## Archive

**[archive/bastion-tunnel-single-tap-auth.md](./archive/bastion-tunnel-single-tap-auth.md)**
A retired technique: reaching an RDP browser appliance through the
bastion with a single hardware-key touch. The appliance it served was
replaced, and the fleet now disables SSH forwarding entirely, so it no
longer applies here. Kept because the difference it explains between a
jump-host tunnel and a jump host that vouches for you is still worth
knowing.

## License

MIT. Everything here is sanitized — no real IP addresses, hostnames,
domains, or vendor-identifying details. Adapt the patterns; the
specifics are placeholders.
