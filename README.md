# Hasler releases

<sub><i>A Delgado &amp; Murphy restoration</i></sub>

Signed software and update channels for the Hasler radios by Delgado & Murphy.

You never need to download anything here: each radio checks for updates by
itself, verifies the signatures and installs only what we signed. The website,
<https://delgado-murphy.github.io/hasler-releases/>, shows the channels and
releases and repeats the radio's signature checks in your browser.

No source code lives here. The code is in
[Delgado-Murphy/hasler](https://github.com/Delgado-Murphy/hasler) (MIT); this
repository only signs finished builds, hosts them as GitHub releases, and
serves the signed channel indexes from GitHub Pages (branch `gh-pages`, written
by the workflow). The site lives at `/hasler-releases/` under the org's
Pages domain; the org site itself (`https://delgado-murphy.github.io/`, repo
`Delgado-Murphy/delgado-murphy.github.io`) must never get a
`hasler-releases/` folder or page of its own: it would conflict with this
site, and every radio's update URL points here.

**Own a Hasler and something is wrong?** Don't open an issue here: on the
radio's key page use **Settings → Something's wrong? → Tell the makers**. It
sends us what we need, and nothing private
([Using your Hasler](https://github.com/Delgado-Murphy/hasler/blob/master/docs/using.md#help-from-the-makers)).

## How a radio uses this

- Once a day (or at once with **Check now**, Settings → This radio →
  Updates) the radio fetches its channel's signed index from GitHub Pages,
  then the release it names, and checks every signature before it touches
  anything (below). Small updates install at night when the owner chose
  Automatic; bigger ones ask first and need a press on the radio.
- Every radio is on Stable unless a maker moved it. The **Update channel**
  picker shows with Maker tools on (tap the version number seven times) or
  when the radio isn't on Stable.
- yt-dlp is updated separately, from `components.json` (below).
- New radios are installed from the `install/` kit on the same site
  (`hasler-firstboot.zip`, or `hasler-install` + `.sig` over SSH), signed
  offline with the root key, namespace `hasler-install`:
  [docs/new-install.md](https://github.com/Delgado-Murphy/hasler/blob/master/docs/new-install.md).
- No makers, no GitHub? The updater also takes `file://`: a copy of this
  site's signed files on a USB stick installs the same way while the index is
  fresh (14 days) and the release under 120 days old
  ([docs/ota.md](https://github.com/Delgado-Murphy/hasler/blob/master/docs/ota.md#5-channels-and-the-fleet)).

## Channels

There are two update channels, always called **Bench** and **Stable** (their
ids in files and workflow inputs are `bench` and `stable`).

| Channel (id) | Who | How a release gets there |
|---|---|---|
| **Bench** (`bench`) | development, unstable: the makers' own radios get every release first; may break. Every release starts here. | `sign-publish` publish - by hand, or by `hasler`'s `auto-release.yml` for automatic dependency updates |
| **Stable** (`stable`) | every other radio (the default) | `sign-publish` promote - automatically after 7 days on Bench (`auto-promote.yml`), or by hand. The first promotion to Stable and every `major` release are done by hand. Until the first one, radios on Stable report "no update info" and keep their version. |

A release moves from Bench to Stable as the *same artifact*, never rebuilt.
What each channel points at today: the [website](https://delgado-murphy.github.io/hasler-releases/)
(it reads the signed indexes). Every automatic promotion is recorded in
[`auto-promote.json`](https://delgado-murphy.github.io/hasler-releases/auto-promote.json)
on gh-pages, newest first.

## What a radio fetches

Base URL: `https://delgado-murphy.github.io/hasler-releases`

| URL | what | signed by (namespace) |
|---|---|---|
| `<base>/keyring.json` + `.sig` | the current release and timestamp public keys | root key (`hasler-keyring`) |
| `<base>/channels/bench.json` + `.sig` | the Bench channel index | timestamp key (`hasler-channel`) |
| `<base>/channels/stable.json` + `.sig` | the Stable channel index (appears with the first promotion) | timestamp key (`hasler-channel`) |
| `https://github.com/Delgado-Murphy/hasler-releases/releases/download/v<version>/release.json` + `.sig` | the release manifest | release key (`hasler-release`) |
| `<base>/components.json` + `.sig` | the yt-dlp builds a radio may install, each a hash-locked wheel list | release key (`hasler-components`) |
| `.../v<version>/hasler-<version>-arm64.tar.zst` | the release tarball | its sha256 is in `release.json` |

`releases.json` and `auto-promote.json` on the same site are informational:
unsigned, and radios never read them.

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
- **promote** is also run by `auto-promote.yml` (below).
- **nightly refresh** - a schedule at 03:17 UTC re-signs every channel index
  with a fresh issued/expires, only if it still verifies. GitHub disables
  schedules in a repository with no activity for 60 days: re-enable it if
  Actions says so, otherwise every radio sees "no fresh update info" 14 days
  later.

`seq` = `(X*10000 + Y*100 + Z)*100 + (N for -rc.N, 99 for a final)`, with Y
and Z below 100: 1.6.0 is `1060099`, 1.6.1-rc.2 is `1060102`. There is no
counter to keep; radios only ever move to a higher seq.

## The components workflow (yt-dlp)

[`.github/workflows/components.yml`](.github/workflows/components.yml), every
6 hours. yt-dlp is not part of a release: YouTube breaks old versions within
weeks, so radios update it on their own, but only to builds listed here.

1. **plan** - PyPI's newest two stable and two nightly (pre-release) yt-dlp
   versions that aren't blocked and aren't listed yet.
2. **build** - per version on a native arm64 runner with Python 3.13 (the
   radio's): a `--require-hashes` lock of `yt-dlp[default]==V` (aarch64 cp313
   wheels, sha256 each), installed in a fresh venv, the radio's offline
   checks, and a best-effort online check: 64 KiB of `bestaudio` of a
   [canary](canary.json) YouTube Music track. Runner IPs are often
   bot-flagged by YouTube, so `tested.online` may be `null`; it never blocks.
3. **publish** (environment `sign`) - the newest two passing builds of each
   channel go into `components.json`, signed with the release key
   (namespace `hasler-components`), verified, pushed to gh-pages. Unchanged,
   it is re-signed once a day; it expires 14 days after signing.

Block a bad version (and unblock it later):

```sh
gh workflow run components.yml -R Delgado-Murphy/hasler-releases -f block=2026.10.21
gh workflow run components.yml -R Delgado-Murphy/hasler-releases -f unblock=2026.10.21
```

[`canary.json`](canary.json) lists the canary tracks - official "Provided to
YouTube" art tracks of old catalogue recordings from three labels. Edit it on
`main`; no code change needed.

## The auto-promote workflow

[`.github/workflows/auto-promote.yml`](.github/workflows/auto-promote.yml),
daily at 06:41 UTC. It promotes Bench's current release to Stable (by
dispatching `sign-publish` mode `promote`) when it has been Bench's release
for 7 days, is not halted, is a final and not `major`, its `release.json` is
valid for 2 more days, and Stable exists with an older release. A newer Bench
release restarts the clock; halting a release stops it for good.

```sh
gh workflow run auto-promote.yml -R Delgado-Murphy/hasler-releases -f action=pause -f reason="why"
gh workflow run auto-promote.yml -R Delgado-Murphy/hasler-releases -f action=resume
gh workflow run auto-promote.yml -R Delgado-Murphy/hasler-releases -f action=check -f dry_run=true
```

Emergency stop: `gh workflow disable auto-promote.yml -R Delgado-Murphy/hasler-releases`.

## Keys and rotation

Three kinds of key (OpenSSH ed25519, `ssh-keygen -Y`):

| key | lives | signs (namespace) |
|---|---|---|
| root | offline only, two USB sticks + paper | `keyring.json` (`hasler-keyring`), the installer (`hasler-install`) |
| release | secret `RELEASE_SIGNING_KEY`, environment `sign` | `release.json` (`hasler-release`), `components.json` (`hasler-components`) |
| timestamp | secret `TIMESTAMP_SIGNING_KEY`, environment `timestamp` | channel indexes (`hasler-channel`) |

Rotating or revoking a release or timestamp key (a leak, or yearly) never
touches the radios' trust root: make a new pair, write a new `keyring.json`
with a higher `keyring_seq` (`tools/keyring.py new --from keyring.json --drop
<old> --key <new>:release:<new>.pub`), sign it offline with root, publish it
to gh-pages, then replace the secret. After a timestamp key change re-sign
each channel once (mode `promote`). The keyring itself expires on
2027-10-20: renew it well before. The full procedure, and the key ceremony:
[docs/release.md, "Key ceremony"](https://github.com/Delgado-Murphy/hasler/blob/master/docs/release.md#key-ceremony).
The root public key: [`trust/`](trust/).

## tools/

Copies of `scripts/release/` (and of `release-tools/hasler-releases/tools/`)
in `hasler` - re-copy them when they change there. All stdlib Python or bash
with `ssh-keygen`.

| file | what |
|---|---|
| `make_manifest.py` | checks a tarball against its `release.json` (size, sha256, members, `files.sha256`) |
| `keyring.py` | creates, signs (offline, root key) and verifies `keyring.json` |
| `channel.py` | edits channel indexes: `promote`, `halt`, `refresh`, `check` |
| `sign.sh` | signs a file with an OpenSSH key (`FILE` -> `FILE.sig`) and verifies the result |
| `verify.sh` | verifies a signature the way a radio does |
| `components.py` | the yt-dlp lane: PyPI, plan, lock, test, assemble, check `components.json` |
| `autopromote.py` | auto-promotion rules, pause/resume, history (`auto-promote.json`) |

## Repo settings this relies on

| what | where | purpose |
|---|---|---|
| `RELEASE_SIGNING_KEY` | secret in environment `sign` | release private key (ed25519, OpenSSH format); signs `release.json` and `components.json` |
| `BUILD_READ_TOKEN` | secret in environment `sign` | fine-grained PAT, owner `Delgado-Murphy`, repo `hasler` only; Actions, Contents, Metadata: read. Expires after a year - renew it in time. Reads the build run, tag and artifact |
| `TIMESTAMP_SIGNING_KEY` | secret in environment `timestamp` | timestamp private key; signs channel indexes |
| `ATTESTATIONS` | repo variable here | `true` turns on the build-attestation check (`hasler` is public and attests its builds) |
| `ARM_RUNNER` | repo variable here | `ubuntu-24.04-arm`: the yt-dlp builds (the default if unset) |
| `trust/root.pub` | this repo, `main` | the root public key from the key ceremony |
| `keyring.json` + `.sig` | `gh-pages` root | the keyring, signed offline by the root key |

Environments `sign` and `timestamp`: no required reviewer, deployment branches
= `main` only. `main` and `gh-pages`: no force push, no deletion. Actions:
GitHub-owned actions only, pinned by full SHA (`actions/checkout`, and for
the yt-dlp lane `actions/setup-python`, `actions/upload-artifact`,
`actions/download-artifact`); the default token is read-only and jobs ask for
`contents: write` (and `auto-promote` for `actions: write`, to dispatch
`sign-publish`) themselves. The schedules (refresh, components,
auto-promote) stop if GitHub disables them after 60 days without activity:
re-enable them if Actions says so.

## Contributing and security

Changes to the workflows and tools start in `hasler` and are copied here:
[CONTRIBUTING.md](CONTRIBUTING.md). A security problem in the signing or
update path: [SECURITY.md](SECURITY.md), not a public issue.

## Licence

MIT - see [LICENSE](LICENSE).

<sub>Delgado &amp; Murphy</sub>
