# trust/

The **public** half of the Hasler root key. Nothing secret ever goes here.

- `root.pub` - `hasler-root-2026`, ed25519, from the key ceremony
  (docs/release.md "Key ceremony" in `Delgado-Murphy/hasler`). Check it with
  `ssh-keygen -lf root.pub`; it must print
  `SHA256:8bc/MCrWk9aekyhNhdtmPtL9jhgB7xcdStuu4G82Ack`.
  A second root key (`root2.pub`) can be added later.
- The same file is installed on every radio as `/etc/hasler/trust/root.pub`.

The workflow uses it to verify `keyring.json` on gh-pages (namespace
`hasler-keyring`) before it trusts any release or timestamp key in it. The root
private key never touches GitHub: it lives encrypted on two offline USB sticks
plus a paper backup and is used only to sign a new `keyring.json`.

Changing a file here changes what the signing workflow trusts - review any
pull request that touches `trust/` as a key change.
