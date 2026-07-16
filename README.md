# updates.dig.net

A single S3 + CloudFront origin serving two independent, side-by-side pull-based update chains
over HTTPS. See [`SPEC.md`](./SPEC.md) for the normative contract and
[`runbooks/`](./runbooks/) for operations.

- **`/v1/<channel>/`** — the signed DIG auto-update **beacon feed** (`delegation.json` +
  `manifest.json`), consumed by the `dig-updater` beacon. The signature is the gate, not the
  transport (#504/#513). Published by `dig-updater`'s `feed.yml`.
- **`/ext/<channel>/`** — the self-hosted **Chrome-extension update chain**: a signed CRX3 +
  an Omaha `updates.xml` per channel, force-installed by a Chromium browser's own updater
  (dig-installer `ExtensionInstallForcelist`). Published by `dig-chrome-extension`'s release
  workflow via GitHub OIDC. See [`runbooks/ext-hosting.md`](./runbooks/ext-hosting.md).
