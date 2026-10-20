# Hasler releases

Signed software and update channels for the Hasler radios by Delgado & Murphy.

You never need to download anything here: each radio checks for updates by
itself, verifies the signatures and installs only what we signed. The website,
<https://delgado-murphy.github.io/hasler-releases/>, shows the channels and
releases and repeats the radio's signature checks in your browser.

No source code lives here. The code is in
[Delgado-Murphy/hasler](https://github.com/Delgado-Murphy/hasler) (MIT); this
repository only signs finished builds, hosts them as GitHub releases, and
serves the signed channel indexes from GitHub Pages (branch `gh-pages`, written
by the workflow).

## Channels

| Channel | Who | Today |
|---|---|---|
| `bench` | our own test radios on the workbench; may break. Every release starts here. | 1.0.1 |
| `stable` | every other radio (the default) | nothing yet: 1.0.0 is promoted after a soak on bench. Until then radios on Stable report "no update info" and keep their version. |

A release moves from bench to stable as the *same artifact*, never rebuilt.

## What a radio fetches

Base URL: `https://delgado-murphy.github.io/hasler-releases`

| URL | what | signed by (namespace) |
|---|---|---|
| `<base>/keyring.json` + `.sig` | the current release and timestamp public keys | root key (`hasler-keyring`) |
| `<base>/channels/bench.json` + `.sig` | the bench channel index | timestamp key (`hasler-channel`) |
| `<base>/channels/stable.json` + `.sig` | the stable channel index (appears with the first promotion) | timestamp key (`hasler-channel`) |
| `https://github.com/Delgado-Murphy/hasler-releases/releases/download/v<version>/release.json` + `.sig` | the release manifest | release key (`hasler-release`) |
| `.../v<version>/hasler-<version>-arm64.tar.zst` | the release tarball | its sha256 is in `release.json` |

`releases.json` on the same site feeds the website only. It is unsigned and
radios never read it.

## How a radio verifies

1. The root public key is installed once in `/etc/hasler/trust/root.pub`
   (the same file as [`trust/root.pub`](trust/), fingerprint
   `SHA256:8bc/MCrWk9aekyhNhdtmPtL9jhgB7xcdStuu4G82Ack`). Its private half is
   offline and never touches GitHub.
2. `keyring.json` is accepted only if a root key signed it, and replaces the
   stored one only if its `keyring_seq` is higher. It lists the release and
   timestamp keys with validity windows.
3. The channel index must be signed by a timestamp key from the keyring and
   not be expired (14 days after issue; re-signed nightly). An expired index
   means "no fresh update info" - nothing installs.
4. `release.json` must be signed by a release key, its sha256 must match the
   index, and its `seq` must be higher than what the radio runs and not
   `halted`. A signed `release.json` is valid for 120 days.
5. The tarball must match the sha256 and size in `release.json`, and every
   file in it must match `files.sha256`. HTTPS is transport only.

The exact formats: [docs/ota-format.md](https://github.com/Delgado-Murphy/hasler/blob/master/docs/ota-format.md).
Why it works this way: [docs/ota.md](https://github.com/Delgado-Murphy/hasler/blob/master/docs/ota.md).

## The sign-publish workflow

[`.github/workflows/sign-publish.yml`](.github/workflows/sign-publish.yml),
run from Actions or with `gh workflow run`. Nobody approves or signs anything
by hand: the keys are secrets of the `sign` and `timestamp` environments, and
only `main` may use them. The full checklist is
[docs/release.md](https://github.com/Delgado-Murphy/hasler/blob/master/docs/release.md).

- **publish** - after `release.yml` in `hasler` has built tag `vX.Y.Z`:

  ```sh
  gh workflow run sign-publish.yml -R Delgado-Murphy/hasler-releases \
    -f mode=publish -f version=X.Y.Z -f artifact_sha256=<sha256> \
    -f run_id=<release.yml run id> -f channel=bench
  ```

  The `publish` job (environment `sign`) re-checks the build run (release.yml,
  push of tag `vX.Y.Z`, green, tag still on the built commit), the artifact
  digest and sha256, the build attestation (if `ATTESTATIONS=true`), the
  tarball against `release.json`, and the keyring against `trust/`. Then it
  signs `release.json` and creates the GitHub release `vX.Y.Z` with the
  tarball, `release.json`, `.sig`, `SHA256SUMS` and the SBOM. `-rc.N`
  versions are pre-releases. The `channel` job (environment `timestamp`)
  points the channel at it, signs the index and pushes `gh-pages`.
- **promote** - the same release to another channel (`-f mode=promote
  -f version=X.Y.Z -f channel=stable`). Only the `channel` job runs. A channel
  only moves to a higher `seq`.
- **halt** - a bad release (`-f mode=halt -f version=X.Y.Z -f channel=...`).
  Its `seq` goes into the channel's `halted` list and the index is re-signed;
  radios drop a staged copy and never install it. Halt it on every channel
  that points at it, then ship the fix as a new, higher version.
- **nightly refresh** - a schedule at 03:17 UTC re-signs every channel index
  with a fresh issued/expires, only if it still verifies. GitHub disables
  schedules in a repository with no activity for 60 days: re-enable it if
  Actions says so, otherwise every radio sees "no fresh update info" 14 days
  later.

`seq` = `(X*10000 + Y*100 + Z)*100 + (N for -rc.N, 99 for a final)`.

## tools/

Copies of `scripts/release/` in `hasler` - re-copy them when they change
there. All stdlib Python or bash with `ssh-keygen`.

| file | what |
|---|---|
| `make_manifest.py` | checks a tarball against its `release.json` (size, sha256, members, `files.sha256`) |
| `keyring.py` | creates, signs (offline, root key) and verifies `keyring.json` |
| `channel.py` | edits channel indexes: `promote`, `halt`, `refresh`, `check` |
| `sign.sh` | signs a file with an OpenSSH key (`FILE` -> `FILE.sig`) and verifies the result |
| `verify.sh` | verifies a signature the way a radio does |

## Repo settings this relies on

| what | where | purpose |
|---|---|---|
| `RELEASE_SIGNING_KEY` | secret in environment `sign` | release private key (ed25519, OpenSSH format); signs `release.json` |
| `BUILD_READ_TOKEN` | secret in environment `sign` | fine-grained PAT, owner `Delgado-Murphy`, repo `hasler` only; Actions, Contents, Metadata: read. Expires after a year - renew it in time. Reads the build run, tag and artifact |
| `TIMESTAMP_SIGNING_KEY` | secret in environment `timestamp` | timestamp private key; signs channel indexes |
| `ATTESTATIONS` | repo variable here | `true` turns on the build-attestation check (`hasler` is public and attests its builds) |
| `trust/root.pub` | this repo, `main` | the root public key from the key ceremony |
| `keyring.json` + `.sig` | `gh-pages` root | the keyring, signed offline by the root key |

Environments `sign` and `timestamp`: no required reviewer, deployment branches
= `main` only. `main` and `gh-pages`: no force push, no deletion. Actions:
GitHub-owned actions only, pinned by full SHA (the workflow uses only
`actions/checkout`); the default token is read-only and jobs ask for
`contents: write` themselves.

## Licence

MIT - see [LICENSE](LICENSE).
