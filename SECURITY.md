# Security

<sub><i>A Delgado &amp; Murphy restoration</i></sub>

This repository decides what every Hasler radio will install, so a hole here
matters more than anywhere else in the project.

## Reporting

Please **don't open a public issue**. Use GitHub's private reporting:
**Security → Report a vulnerability** on this repository (or on
[`hasler`](https://github.com/Delgado-Murphy/hasler/security) for the
radio's software and updater). We answer within a week, and say when a fix
is out.

Worth a report: a way to make a radio accept a file we didn't sign, roll
back to an older release, accept an expired index or keyring, or a way to
use the signing secrets from anything but `main`. Also: a leaked key or
token anywhere.

## What protects the radios

- Each radio trusts only the offline root key (`trust/root.pub`); the
  release and timestamp keys come from a keyring that root signed.
- Releases, channel indexes and `components.json` are signed and expire;
  `seq` only goes up; a halted release never installs.
- The signing secrets are in environments that only `main` may use.

The threat model in full:
[docs/ota.md, "Security"](https://github.com/Delgado-Murphy/hasler/blob/master/docs/ota.md#3-security).
What we do when a key leaks:
[docs/release.md, "Key ceremony"](https://github.com/Delgado-Murphy/hasler/blob/master/docs/release.md#key-ceremony)
(rotation or revocation) and `sign-publish` mode `halt` for a bad release.

<sub>Delgado &amp; Murphy</sub>
