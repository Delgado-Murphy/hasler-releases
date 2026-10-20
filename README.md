# Hasler releases

Signed release artifacts and update channels for the Hasler radios by Delgado & Murphy.
No source code lives here (it is in the private `Delgado-Murphy/hasler`); this
repo only signs finished builds, hosts them as GitHub releases, and serves the
signed channel indexes from GitHub Pages.

## What a radio fetches

Base URL: `https://delgado-murphy.github.io/hasler-releases`

| URL | what | signed by (namespace) |
|---|---|---|
| `<base>/keyring.json` + `.sig` | the current release and timestamp public keys | root key (`hasler-keyring`) |
| `<base>/channels/bench.json` + `.sig` | owner's bench box | timestamp key (`hasler-channel`) |
| `<base>/channels/beta.json` + `.sig` | 1-2 family boxes | timestamp key (`hasler-channel`) |
| `<base>/channels/stable.json` + `.sig` | everyone | timestamp key (`hasler-channel`) |
| `https://github.com/Delgado-Murphy/hasler-releases/releases/download/v<version>/release.json` + `.sig` | the release manifest | release key (`hasler-release`) |
| `.../v<version>/hasler-<version>-arm64.tar.zst` | the release tarball | its sha256 is in `release.json` |

The channel files and the keyring appear with the first signed release; until
then the Pages site has only a placeholder.

## How a radio verifies

1. Root public key(s) are installed once in `/etc/hasler/trust/root*.pub`
   (same files as `trust/` here).
2. `keyring.json` is accepted only if a root key signed it; a new one
   replaces the stored one only if its `keyring_seq` is higher. It lists the
   release and timestamp keys with validity windows.
3. The channel index must be signed by a timestamp key from the keyring and
   not be expired (indexes expire 14 days after issue; this repo re-signs them
   daily). An expired index means "no fresh update info" - nothing installs.
4. `release.json` must be signed by a release key, its sha256 must match the
   index, its `seq` must be higher than what the box runs and not in `halted`.
5. The tarball must match the sha256/size in `release.json`, and every file in
   it must match `files.sha256`. HTTPS is transport only.

Formats: `docs/ota-format.md` in the private repo. The scripts in `tools/` are
copies of its `scripts/release/` (`make_manifest.py`, `keyring.py`,
`channel.py`, `sign.sh`, `verify.sh`) - re-copy them when they change there.

## Publishing a release

1. Private repo: push tag `vX.Y.Z-rc.N` (or `vX.Y.Z`). `release.yml` tests,
   builds reproducibly on arm64 and uploads the artifact
   `hasler-release-<version>`; its run summary shows the **run id** and the
   tarball's **sha256**. Check the artifact on the laptop (docs/release.md step 5).
2. Here: Actions -> **sign-publish** -> Run workflow: mode `publish`,
   version (no `v`), artifact sha256, run id, channel (normally `bench`).
3. The `publish` job waits for **review on the `sign` environment** - approve
   it. It re-checks the run (release.yml, push of tag `v<version>`, green, tag
   still on the built commit), the artifact digest, the sha256, the
   attestation (if `ATTESTATIONS=true`), the tarball against `release.json`,
   the keyring against `trust/`; then signs `release.json` and creates the
   GitHub release `v<version>` with the tarball, `release.json`, `.sig`,
   `SHA256SUMS` (and the SBOM if present). `-rc.N` versions are pre-releases.
4. The `channel` job (environment `timestamp`) points the channel at the new
   release, signs the index and pushes `gh-pages`.

## Promote, halt, refresh

- **Promote** (same artifact, never rebuilt): sign-publish with mode
  `promote`, the version, and the target channel (`beta` / `stable`). Only
  the `channel` job runs. A channel only moves to a higher `seq`.
- **Halt** a bad release: sign-publish with mode `halt`, the bad version and
  the channel. Its `seq` is added to the channel's `halted` list and the
  index is re-signed; boxes drop a staged copy and never install it. Halt it
  on every channel that points at it. Then ship the fix as a new, higher
  version (a "pull back" is also a new, higher-seq release).
- **Refresh**: a daily schedule (03:17 UTC) re-signs every channel index with
  a fresh issued/expires, only if it still verifies. GitHub disables schedules
  in a repo with no activity for 60 days - re-enable it if Actions says so,
  otherwise all boxes see "no fresh update info" after 14 days.

`seq` = `(M*10000 + m*100 + p)*100 + (N for rc.N, 99 for a final)`.

## Repo settings this relies on

| what | where | purpose |
|---|---|---|
| `RELEASE_SIGNING_KEY` | secret in environment `sign` | release private key (ed25519, unencrypted OpenSSH format); signs `release.json` |
| `BUILD_READ_TOKEN` | secret in environment `sign` | fine-grained PAT, owner `Delgado-Murphy`, repo `hasler` only; Actions: read, Contents: read, Metadata: read; 1-year expiry (calendar it). Reads the build run, tag and artifact |
| `TIMESTAMP_SIGNING_KEY` | secret in environment `timestamp` | timestamp private key; signs channel indexes |
| `ATTESTATIONS` | repo variable (optional) | `true` only when the private repo can produce build attestations (Enterprise Cloud); unset = rely on run metadata + artifact digest |
| `trust/root.pub` (`root2.pub`) | this repo, `main` | root public key(s) from the key ceremony |
| `keyring.json` + `.sig` | `gh-pages` root | first keyring from the key ceremony |

Environment `sign`: required reviewer = owner; `timestamp`: no reviewer (the
daily job runs alone). Both: deployment branches = `main` only. Actions:
GitHub-owned actions only, pinned by full SHA (the workflow uses only
`actions/checkout`); default token read-only - jobs ask for `contents: write`
themselves. The `github-actions` app must be able to create `v*` releases/tags
and push `gh-pages` (if a tag ruleset is added, give it a bypass).
