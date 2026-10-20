# trust/

The **public** halves of the Hasler root keys. Nothing secret ever goes here.

- `root.pub` (and optionally `root2.pub`) - ed25519 OpenSSH public keys, added
  at the key ceremony (docs/release.md "Key ceremony" in the private repo).
  Until then this directory holds only this README, and every signing job fails
  on purpose at the keyring check.
- The same files are installed on every radio as `/etc/hasler/trust/root*.pub`.

The workflow uses them to verify `keyring.json` on gh-pages (namespace
`hasler-keyring`) before it trusts any release or timestamp key in it. The root
private keys never touch GitHub: they live encrypted on two offline USB sticks
plus a paper backup and are used only to sign a new `keyring.json`.

Changing a file here changes what the signing workflow trusts - review any
pull request that touches `trust/` as a key change.
