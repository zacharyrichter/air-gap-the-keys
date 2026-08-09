# Note: Why Repository-Level Encryption Isn't Used Here

Hardware configuration backups (network device configs — switch,
access point, router — which can contain things like local admin
credentials and pre-shared keys) are stored in a private,
access-controlled Git repository. They are **not** encrypted at rest
within that repository. This was a deliberate decision, not an
oversight, and it's worth documenting the reasoning rather than just
the outcome.

## What was tried

Transparent, repository-level encryption via `git-crypt` — a tool that
wires an encrypt/decrypt filter into Git's own `clean`/`smudge`
mechanism, so `git add`/`commit` encrypts file content automatically and
the repository host never sees plaintext. It was fully implemented and
verified working end to end: a fresh clone showed genuine ciphertext
before unlocking, and correct plaintext after unlocking with the actual
offline backup copy of the decryption key — not just the working copy
still sitting on the machine that set it up.

## Why it was reverted anyway

One device in this setup pushes its own configuration backup
**automatically**, using its own internal Git client, running on its
own separate operating system. That client has no way to participate in
a Git filter that only exists in a `.git/config` on a different machine
— the filter is local configuration, not something a repository can
enforce on every writer touching it.

The result would have been a repository where every *manually*-managed
config was genuinely encrypted, and the one *automatically*-managed
config kept landing in plaintext, indefinitely, with no way to fix that
from the automation side without replacing how that device backs up its
config entirely (real, but disproportionate, additional work).

That's a worse state than the one it was meant to improve. A repository
that's honestly, consistently unencrypted is easy to reason about: you
know exactly what protects it (access control, and nothing else). A
repository that's *mostly* encrypted looks more secure at a glance than
it actually is — and a false sense of a stronger security posture is a
more dangerous failure mode than an accurately-understood weaker one,
because it changes how carefully everything downstream treats that
repository's contents.

## The principle this reinforces

This ties back to the same design rule used throughout the rest of the
project: a security control that only partially applies is often worse
than no control at all, because it obscures where the actual boundary
is. The repository stayed private and access-controlled — a real,
understood protection — rather than gaining an inconsistent one that
would have been easy to mistake for something stronger than it was.

If the automated device's backup mechanism is ever replaced with
something that *can* participate in local encryption (a custom push
script instead of vendor firmware doing its own Git operations), this
decision is worth revisiting. Until then, the private repository is the
whole security boundary, by design, not by omission.
