# updates.dig.net — normative hosting contract

`updates.dig.net` is a single S3 + CloudFront origin that serves two INDEPENDENT,
side-by-side update chains. Both are pull-based over plain HTTPS: the signature (beacon)
or the CRX3 id-binding (extension) is the trust gate, never the transport.

- **`/v1/<channel>/…`** — the signed DIG binary-release **beacon feed** (delegation.json +
  manifest.json), consumed by the `dig-updater` beacon. Published by `dig-updater`'s
  `feed.yml`. Unaffected by the extension chain below.
- **`/ext/<channel>/…`** — the self-hosted **Chrome-extension update chain** (this document):
  a signed CRX3 + an Omaha `updates.xml` per channel, consumed by a Chromium browser's own
  extension updater via an `ExtensionInstallForcelist` policy (dig-installer #612).

Channels: `nightly` (mandatory self-hosted) and `stable` (self-hosted; Chrome Web Store is an
optional future alternative — see #602 D1).

## 1. Infrastructure (AWS account 873139760123, us-east-1)

| Resource | Identifier |
|---|---|
| S3 bucket (private, CloudFront OAC) | `updates-dig-net` |
| CloudFront distribution | `E1J7P08GC3CELV` (alias `updates.dig.net`) |
| CloudFront cache policy for `/ext/*` | `updates-ext-assets` — `c6e92015-fd84-49f4-b834-09c79eec9fdc` |
| Beacon-feed OIDC publish role | `updates-dig-net-deploy` (dig-updater / this repo) |
| Extension OIDC publish role | `updates-dig-net-ext-deploy` — `arn:aws:iam::873139760123:role/updates-dig-net-ext-deploy` |

The `/ext/*` CloudFront behavior targets the same S3 origin as the default behavior and uses the
`updates-ext-assets` cache policy: `MinTTL=0, DefaultTTL=300, MaxTTL=31536000`, honoring the
object's origin `Cache-Control`. This lets one behavior serve BOTH an immutable long-TTL `.crx`
and a short-TTL `updates.xml` — the per-object `Cache-Control` (§3) is authoritative; the
`DefaultTTL=300` is a safe failsafe if an object is ever uploaded without one.

## 2. S3 layout (per channel)

```
s3://updates-dig-net/ext/nightly/updates.xml           # Omaha gupdate manifest (mutable pointer)
s3://updates-dig-net/ext/nightly/dig-ext-<VERSION>.crx  # immutable, version-named signed CRX3
s3://updates-dig-net/ext/stable/updates.xml
s3://updates-dig-net/ext/stable/dig-ext-<VERSION>.crx
```

`<VERSION>` is the CRX manifest version (dot-separated integers, each 0–65535 — Omaha requirement;
the nightly scheme is `maj.min.patch.<days-since-2020-01-01>`, #607/#602 D2). CRX filenames are
**version-named and immutable** — a new build is a NEW object, never an overwrite.

## 3. Served responses (CloudFront)

| Object | `Content-Type` | `Cache-Control` | Rationale |
|---|---|---|---|
| `*.crx` | `application/x-chrome-extension` | `public, max-age=31536000, immutable` | version-named, never changes → cache forever |
| `updates.xml` | `text/xml; charset=utf-8` | `public, max-age=60, must-revalidate` | mutable pointer → every browser re-polls within ~1 min so a new nightly propagates across the fleet the same day |

`https://updates.dig.net/ext/<channel>/updates.xml` is the **`update_url`** compiled into the
`ExtensionInstallForcelist` policy (dig-installer). It MUST remain stable forever.

### `updates.xml` shape (Omaha gupdate protocol 2.0)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<gupdate xmlns="http://www.google.com/update2/response" protocol="2.0">
  <app appid="mlibddmbhlgogepnjdienclhnkfpkfah">
    <updatecheck codebase="https://updates.dig.net/ext/nightly/dig-ext-<VERSION>.crx"
                 version="<VERSION>"/>
  </app>
</gupdate>
```

- `appid` = the channel's stable extension id (`SHA256`-derived from the signing-key SPKI).
  Nightly = `mlibddmbhlgogepnjdienclhnkfpkfah` (pinned in SYSTEM.md; MUST NOT drift).
- `codebase` = the absolute HTTPS URL of the immutable CRX for `version`.
- A `<gupdate>` with NO `<app>` (or an `<app>` with no `<updatecheck>`) is the valid
  **"no update available"** state — the initial/seeded state before any CRX is published.

## 4. Publish invariants (HARD)

The extension repo's release workflow (`DIG-Network/dig-chrome-extension`) publishes here by
assuming `updates-dig-net-ext-deploy` via GitHub OIDC. The publish MUST be:

1. **CRX-first, manifest-last.** Upload the immutable `dig-ext-<VERSION>.crx` and CONFIRM it is
   fetchable (`HEAD 200`) BEFORE writing/overwriting `updates.xml`. This guarantees `updates.xml`
   never points at a missing CRX (fail-loud: abort the whole publish if the CRX upload fails).
2. **Correct per-object metadata** — the `Content-Type` + `Cache-Control` of §3, set at upload
   (`--content-type` / `--cache-control`).
3. **Invalidate only the pointer.** After a successful publish, create a CloudFront invalidation
   for `/ext/<channel>/updates.xml` (never the immutable CRXs — they are version-named and never
   change, so invalidating them is waste).
4. **Scope-bounded.** The role can write ONLY under `ext/*` and invalidate on this distribution —
   it CANNOT touch the beacon `/v1/` feed (§5).

## 5. OIDC publish role — `updates-dig-net-ext-deploy`

Least-privilege, scoped to the extension chain only (never the beacon feed):

- **Trust** (GitHub OIDC, `token.actions.githubusercontent.com`):
  `repo:DIG-Network/dig-chrome-extension:ref:refs/heads/main`,
  `repo:DIG-Network/dig-chrome-extension:ref:refs/tags/*`,
  `repo:DIG-Network/dig-chrome-extension:environment:release`
  (aud `sts.amazonaws.com`).
- **Permissions** (inline `updates-dig-net-ext-publish`):
  - `s3:PutObject` on `arn:aws:s3:::updates-dig-net/ext/*`
  - `s3:ListBucket` on `arn:aws:s3:::updates-dig-net` (condition `s3:prefix ∈ {ext/*, ext}`)
  - `cloudfront:CreateInvalidation` on `distribution/E1J7P08GC3CELV`

The role ARN + bucket + distribution id are exposed to the extension repo as the CI variables
`UPDATES_EXT_DEPLOY_ROLE`, `UPDATES_BUCKET`, `UPDATES_CF_DISTRIBUTION_ID`.
