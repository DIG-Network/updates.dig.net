# Runbook — extension CRX + updates.xml hosting (`/ext/`)

Normative contract: `SPEC.md`. This runbook is the operational how-to for the extension update
chain served at `https://updates.dig.net/ext/<channel>/`.

## What is hosted

- `ext/<channel>/dig-ext-<VERSION>.crx` — the signed CRX3 (immutable, version-named).
- `ext/<channel>/updates.xml` — the Omaha gupdate manifest pointing at the current CRX.

Channels: `nightly`, `stable`. Both live beside the beacon feed at `/v1/<channel>/` (untouched).

## Who publishes

`DIG-Network/dig-chrome-extension`'s release workflow, via GitHub OIDC assuming
`arn:aws:iam::873139760123:role/updates-dig-net-ext-deploy`. No long-lived AWS keys.

CI variables already set on the extension repo:

| Variable | Value |
|---|---|
| `UPDATES_EXT_DEPLOY_ROLE` | `arn:aws:iam::873139760123:role/updates-dig-net-ext-deploy` |
| `UPDATES_BUCKET` | `updates-dig-net` |
| `UPDATES_CF_DISTRIBUTION_ID` | `E1J7P08GC3CELV` |

## Publish sequence (the fail-loud contract)

The extension workflow, after packaging the signed CRX (#607):

```bash
# 1. CRX first — immutable, cache-forever, correct content-type
aws s3 cp "dig-ext-${VERSION}.crx" "s3://${UPDATES_BUCKET}/ext/${CHANNEL}/dig-ext-${VERSION}.crx" \
  --content-type "application/x-chrome-extension" \
  --cache-control "public, max-age=31536000, immutable"

# 2. Confirm the CRX is actually there before pointing at it
aws s3api head-object --bucket "${UPDATES_BUCKET}" --key "ext/${CHANNEL}/dig-ext-${VERSION}.crx"

# 3. Manifest last — short TTL, correct content-type
aws s3 cp "updates.xml" "s3://${UPDATES_BUCKET}/ext/${CHANNEL}/updates.xml" \
  --content-type "text/xml; charset=utf-8" \
  --cache-control "public, max-age=60, must-revalidate"

# 4. Invalidate ONLY the mutable pointer (never the immutable CRXs)
aws cloudfront create-invalidation --distribution-id "${UPDATES_CF_DISTRIBUTION_ID}" \
  --paths "/ext/${CHANNEL}/updates.xml"
```

Rules: never write `updates.xml` before the CRX it references is confirmed fetchable; a failed CRX
upload aborts the whole publish (no stale pointer). CRXs are version-named — never overwrite one.

## Verify a publish went live

```bash
curl -sSI https://updates.dig.net/ext/nightly/updates.xml   # 200, text/xml, max-age=60
curl -sS  https://updates.dig.net/ext/nightly/updates.xml   # <updatecheck codebase=... version=...>
# fetch the CRX the manifest points at:
curl -sSI "$(curl -sS https://updates.dig.net/ext/nightly/updates.xml \
  | grep -o 'codebase="[^"]*"' | cut -d'"' -f2)"            # 200, application/x-chrome-extension, immutable
```

End-to-end: a browser with an `ExtensionInstallForcelist` entry
`mlibddmbhlgogepnjdienclhnkfpkfah;https://updates.dig.net/ext/nightly/updates.xml`
force-installs the current nightly, then auto-updates when a higher-versioned CRX is published.

## Infra (provisioned via the `aws` skill, idempotent, tag `managed-by=dig-loop`)

- CloudFront `/ext/*` behavior on `E1J7P08GC3CELV` → S3 origin, cache policy `updates-ext-assets`
  (`c6e92015-fd84-49f4-b834-09c79eec9fdc`; MinTTL 0 / DefaultTTL 300 / MaxTTL 1y, honors origin
  `Cache-Control`).
- IAM role `updates-dig-net-ext-deploy` + inline policy `updates-dig-net-ext-publish` (see SPEC §5).

To re-inspect:

```bash
aws cloudfront get-distribution-config --id E1J7P08GC3CELV \
  --query 'DistributionConfig.CacheBehaviors.Items[?PathPattern==`/ext/*`]'
aws iam get-role-policy --role-name updates-dig-net-ext-deploy --policy-name updates-dig-net-ext-publish
```
