# Contributing

<sub><i>A Delgado &amp; Murphy restoration</i></sub>

This repository signs and serves Hasler releases. It holds no source code,
and most of what is here is a copy, so most changes belong elsewhere.

**Own a radio and something is wrong?** Use **Settings → Something's wrong?
→ Tell the makers** on the radio's key page instead of an issue here.

## Where a change goes

| You want to change | Do it in | Then here |
|---|---|---|
| the radio's software, the updater, the file formats | [`hasler`](https://github.com/Delgado-Murphy/hasler) | nothing - a release brings it |
| `tools/*.py`, `tools/*.sh` | `hasler`: `scripts/release/` and `release-tools/hasler-releases/tools/` | re-copy the files, one PR |
| the workflows in `.github/workflows/`, `canary.json` | `hasler`: `release-tools/hasler-releases/` | re-copy, one PR |
| `canary.json` only (a dead canary track) | here, on `main` | - |
| `trust/` | only after a key ceremony | review it as a key change |
| this README, CONTRIBUTING, SECURITY | here | - |

Never commit a private key, a token, or anything under `gh-pages`: the
workflows write that branch, and channel indexes, `keyring.json`, releases
and `components.json` change only through them.

## Checking a change locally

Everything is stdlib Python 3 or bash with `ssh-keygen`; no install step.

```sh
python3 tools/channel.py --help          # every tool has --help
# check today's Bench channel the way a radio does
B=https://delgado-murphy.github.io/hasler-releases
for f in keyring.json keyring.json.sig channels/bench.json channels/bench.json.sig; do curl -fsSLO "$B/$f"; done
python3 tools/keyring.py verify --root trust/root.pub keyring.json
bash tools/verify.sh -n hasler-channel -r timestamp -K keyring.json bench.json
```

The tests for these tools live in `hasler` (`scripts/release/`,
`release-tools/`) and run in its CI. To check what is live, the website
(<https://delgado-murphy.github.io/hasler-releases/>) re-verifies every
signature in the browser, and
`gh workflow run auto-promote.yml -f action=check -f dry_run=true` says what
auto-promotion would do.

## Pull requests

Small and one topic each. Say which `hasler` commit a copied file comes
from. Workflow changes keep to GitHub-owned actions pinned by full SHA, a
read-only default token, and `main` as the only branch that may sign.

The release checklist itself:
[docs/release.md](https://github.com/Delgado-Murphy/hasler/blob/master/docs/release.md).

<sub>Delgado &amp; Murphy</sub>
