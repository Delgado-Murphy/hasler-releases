# Hasler releases

Signed release artifacts and update channels for the Hasler radios by Delgado & Murphy.
No source code lives here. Every release is signed (OpenSSH ed25519, namespace `hasler-release`);
radios verify against a keyring signed by an offline root key and refuse anything else.

Channel indexes are served from GitHub Pages: `channels/<bench|beta|stable>.json`.
