# Chrome Web Store acquisition: decision and contract

_Checked 2026-07-30. This is product/engineering research, not legal advice._

## Decision

The server-side acquisition flow is **technically viable**, but the intended
anonymous “analyze any Chrome Web Store URL” launch is **not cleared by the
published first-party terms**.

Launch the MVP with a **publisher-authorized allowlist**, beginning with FX
Inline. Each allowlisted extension must have a retained record that its
publisher authorized server-side package retrieval and analysis. Expand beyond
that allowlist only after either:

1. Google gives written permission for this server-side use of the Web Store
   update/item services and the publisher has authorized the analysis; or
2. counsel confirms a different permission basis for the target jurisdictions
   and for any extension-specific EULA.

A checkbox from an anonymous submitter is not publisher authorization. The
website may keep the Chrome Web Store URL input, but it must reject
non-allowlisted IDs before calling Google in the initial release.

Why the gate is necessary:

- The Chrome Web Store Developer Agreement says access must use an interface
  Google provides unless Google separately permits another means, and retains
  third-party intellectual-property protections ([sections 4.4 and
  5](https://developer.chrome.com/docs/webstore/program-policies/terms)).
- The general Google Terms prohibit automated access that violates
  machine-readable instructions and say third-party content may not be used
  without its owner's permission or another legal basis ([Google
  Terms](https://policies.google.com/terms)).
- A Web Store publisher grants users a license to perform, display, and use the
  product **in connection with Google Chrome**, but may replace that grant with
  its own EULA ([Developer Agreement, section
  5.2](https://developer.chrome.com/docs/webstore/program-policies/terms)).
  The text does not clearly grant an unrelated public service the right to copy
  every publisher's package onto its servers for static analysis.
- The documented Chrome Web Store API is an authenticated publisher-management
  API; it exposes upload/status/publish operations for items owned by the
  authenticated publisher, not a documented public package-download method
  ([API guide](https://developer.chrome.com/docs/webstore/using-api),
  [REST reference](https://developer.chrome.com/docs/webstore/api/reference/rest)).

Robots rules are necessary but not sufficient permission. The current Web Store
robots file permits a base `/detail/...` page but disallows several subroutes
and search ([robots.txt](https://chromewebstore.google.com/robots.txt)). This
design does not scrape listing HTML.

## What first-party evidence establishes

Chrome documents `https://clients2.google.com/service/update2/crx` as the Chrome
Web Store update URL and says the extension ID is present in the Web Store URL
([alternative installation
methods](https://developer.chrome.com/docs/extensions/how-to/distribute/install-extensions)).
Current Chromium source:

- constructs an update check as repeated URL-encoded
  `x=id=<ID>&v=<VERSION>&uc` values
  ([`manifest_fetch_data.cc`](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/extensions/browser/updater/manifest_fetch_data.cc));
- identifies the same Web Store update URL
  ([`extension_urls.cc`](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/extensions/common/extension_urls.cc));
- receives version, download URL, expected SHA-256, and size in the updater
  protocol ([updater protocol](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/updater/protocol_3_1.md));
- verifies Web Store packages as CRX3 with a production publisher proof
  ([`verifier_formats.cc`](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/extensions/common/verifier_formats.cc),
  [`crx_verifier.h`](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/components/crx_file/crx_verifier.h)).

The official VersionHistory API provides a programmatic current Chrome version
for a chosen platform and Stable channel
([guide](https://developer.chrome.com/docs/web-platform/versionhistory/guide),
[reference](https://developer.chrome.com/docs/web-platform/versionhistory/reference)).

Chromium also contains an item-snippet endpoint and states that it does not
require an API key
([`webstore_data_fetcher.cc`](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/extensions/browser/webstore_data_fetcher.cc)).
It is useful as a diagnostic, but it is not part of the documented public Web
Store REST surface and must not be a launch dependency.

### Reproducible observation: FX Inline

On 2026-07-30, using extension ID
`cicoldboionelecmpjpkkdhicjbpfefj`:

- the first-party item-snippet endpoint returned HTTP 200 with the matching
  `itemId` and manifest version `0.5.0`;
- the first-party update check returned HTTP 200 with version `0.5.0`, size
  `83,572`, SHA-256
  `41cc20a8ef65d80d4a190bf149af2ab07e35e33f638db83e2db67c4e56a0de10`,
  and an HTTPS `clients2.googleusercontent.com` CRX URL;
- the CRX object returned
  `Content-Type: application/x-chrome-extension` and `Content-Length: 83572`;
- the downloaded entity was 83,572 bytes, its SHA-256 matched, its envelope
  began with `Cr24` and CRX version 3, and its root `manifest.json` declared
  `name: "FX Inline"` and `version: "0.5.0"`;
- a syntactically valid but unavailable ID returned HTTP 404 from the update
  endpoint.

These are observations, not an SLA or a promise that the endpoints or response
shape will remain unchanged. Google expressly provides services without
availability or reliability warranties ([Google
Terms](https://policies.google.com/terms)).

## Acquisition contract

This contract applies only after the permission gate above succeeds.

### 1. Parse and canonicalize the submitted URL locally

Use a standards-compliant URL parser. Never request, resolve DNS for, or follow
a redirect from the submitted URL.

Accept:

- scheme: `https`;
- origin: exactly `https://chromewebstore.google.com`, or the legacy exact
  origin `https://chrome.google.com` with path beginning `/webstore/detail/`;
- no username, password, non-default port, or percent-encoded host;
- path shape:
  `/detail/<id>`, `/detail/<slug>/<id>`,
  `/webstore/detail/<id>`, or `/webstore/detail/<slug>/<id>`, with at most one
  trailing slash;
- an ID whose lowercase canonical form matches `^[a-p]{32}$`.

Chromium derives a 32-character ID from the first 16 bytes of a SHA-256 public
key hash and maps hexadecimal digits to `a` through `p`
([`id_util.cc`](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/components/crx_file/id_util.cc)).

Ignore the slug, query, and fragment after successful parsing; never use them in
a cache key or upstream request. Reject extra path segments, encoded slashes,
control characters, invalid UTF-8, lookalike hosts, and all bare IDs. Emit the
canonical reference:

```text
https://chromewebstore.google.com/detail/<lowercase-id>
```

### 2. Apply permission and local abuse gates

Before any Google request:

1. require the canonical ID to be on the publisher-authorized allowlist;
2. enforce the product's three accepted, uncached distinct IDs per IP per
   rolling hour rule;
3. coalesce concurrent work for the same ID into one acquisition (single
   flight);
4. check a short-lived `current_release` pointer so repeated submissions do not
   cause needless update checks.

Cached report views and same-version hits do not consume the distinct-extension
allowance. Invalid and unauthorized input uses a separate inexpensive abuse
throttle.

### 3. Define “current” explicitly

The Web Store supports staged and percentage rollouts, so there may not be one
globally served package at an instant ([update
documentation](https://developer.chrome.com/docs/webstore/update/)).

For this product, **current release** means:

> The full CRX returned at acquisition time by the Chrome Web Store update
> service for a stateless Linux x86-64, Stable Chrome client version selected
> from the official VersionHistory API.

Refresh and pin that Stable Chrome version daily. Record the client version,
platform, acquisition timestamp, extension version, and CRX hash with the
report. Do not call a partially rolled version “the version all users receive.”

### 4. Resolve the release through an update check

Send one credential-free HTTPS GET to the exact update endpoint:

```text
https://clients2.google.com/service/update2/crx
```

Use a fixed, server-built parameter set equivalent to Chromium's update-check
shape. Set the requested installed version to `0.0.0.0`, request CRX3, identify
the fixed Linux/x86-64 environment, and include exactly one URL-encoded
`x=id=<canonical-id>&v=0.0.0.0&uc` value. Do not accept arbitrary query values
from the visitor. Do not send cookies, Chrome API keys, Chrome-only
traffic-management headers, or a misleading Chrome user agent. Identify the
service honestly and include an operator contact where HTTP conventions allow.

Parse at most 256 KiB of response XML with DTDs, external entities, and network
resolution disabled. Require:

- HTTP 200;
- one `app` whose `appid` exactly equals the canonical ID;
- successful `app` and `updatecheck` status;
- a valid Chrome extension `version`;
- integer `size >= 0`;
- exactly 64 lowercase hexadecimal characters in `hash_sha256`;
- a cryptographic HTTPS `codebase` URL on the exact host
  `clients2.googleusercontent.com`.

Chrome versions are one to four dot-separated integers with each component from
0 through 65535
([manifest version](https://developer.chrome.com/docs/extensions/reference/manifest/version)).
Reject rather than guess if a required field is missing or ambiguous.

### 5. Enforce the 30 MB download boundary twice

Define the UI's “30 MB” as **30,000,000 bytes of CRX entity body**, including
the CRX envelope.

1. **Preflight:** if update metadata `size` exceeds 30,000,000, return
   `package_too_large` without fetching the CRX.
2. **Streaming:** request `Accept-Encoding: identity`; reject an advertised
   `Content-Length` over the cap; write to a fresh ephemeral file while
   incrementally hashing and counting bytes; abort as soon as byte
   30,000,001 arrives.
3. Require final byte count to equal both update metadata `size` and any
   `Content-Length`, and require the SHA-256 to equal `hash_sha256`.

HEAD is only an optimization and never the sole size check.

### 6. Constrain redirects and validate the package

Prefer downloading the validated update-manifest `codebase` directly. Follow no
more than one redirect, and only after re-validating HTTPS, the exact
`clients2.googleusercontent.com` host, default port, and a `/crx/` path. Reject
cross-host, downgrade, credential-bearing, IP-literal, and excess redirects.

Before DocX receives the file:

1. require a `Cr24` envelope and CRX version 3;
2. verify the complete CRX3 signature and the production Web Store publisher
   proof using the current Chromium verification semantics;
3. derive the CRX ID from the verified key/proof and require exact equality with
   the canonical submitted ID;
4. locate exactly one root `manifest.json`, parse it under a small byte limit,
   and require its `version` to equal update metadata;
5. compute the immutable release identity
   `{store: "chrome-web-store", extension_id, version, crx_sha256}`.

Content type is supporting evidence only; it is not a substitute for signature,
ID, hash, and envelope verification.

Hand only that verified ephemeral path plus typed release metadata to the
isolated analyzer worker. Never execute extension JavaScript. Extraction byte,
file-count, path, CPU, memory, and time limits belong to the worker-sandbox
contract and remain mandatory.

### 7. Cache without rerunning the same release

Use `(extension_id, version, crx_sha256)` as the report key and
`(extension_id, version)` as a uniqueness assertion:

- existing matching hash: return the sanitized cached report and skip DocX;
- no match: atomically create one analysis job and have all duplicate requests
  observe it;
- same ID/version but different hash: quarantine both observations as an
  integrity incident; never overwrite or silently rerun.

Web Store uploads must increase the version, and rollback republishes old
contents under a new version number
([update](https://developer.chrome.com/docs/webstore/update/),
[rollback](https://developer.chrome.com/docs/webstore/rollback)). A same-version
hash change is therefore not a normal update.

Delete the CRX and extracted files immediately after successful or failed
analysis. Retain only the sanitized report, typed release metadata, aggregate
timings, and the existing 24-hour failed-job record.

### 8. Handle publication races

The update response's version, size, hash, and codebase are one snapshot. The
downloaded manifest and verified bytes define what was actually analyzed.

- If the hash, size, ID, or manifest version disagrees, discard the file,
  refresh the update check once with jitter, and retry once.
- If the second snapshot is inconsistent, return `upstream_inconsistent`; do not
  analyze either file.
- If a newer release appears after a verified acquisition begins, finish and
  label the exact acquired release. A later submission can advance the
  `current_release` pointer; it must not invalidate the older immutable report.
- Recheck upstream only after the short current-pointer TTL. Rechecking metadata
  is not rerunning DocX.

## Status and retry matrix

| Condition | Result | Retry behavior |
|---|---|---|
| Invalid/non-Web-Store URL | `invalid_store_url` | Never upstream; do not retry |
| Valid ID without publisher authorization | `extension_not_authorized` | Never upstream |
| Item/update says no app, no update, 404, 410, or permission denied | `extension_unavailable` | Do not claim removed vs private, unlisted, geo-restricted, or policy-blocked; retain failed job per product policy |
| Unlisted item whose direct ID resolves and is authorized | Continue | Unlisted status alone is not an error |
| Update metadata size over 30,000,000 | `package_too_large` | Do not download |
| Download crosses cap | `package_too_large` | Abort immediately; no automatic retry |
| 429 | `upstream_rate_limited` | Honor bounded `Retry-After`; otherwise fail the job |
| Network error, 408, or 5xx | `upstream_transient` | At most two retries with exponential backoff and jitter |
| Other 4xx | `upstream_rejected` | No automatic retry |
| Hash/size/ID/version mismatch | `upstream_inconsistent` | One fresh-snapshot retry, then quarantine |
| Invalid CRX3/publisher proof | `invalid_store_package` | Quarantine; no analyzer run |
| Matching cached release | `cache_hit` | No CRX download or analyzer run while the current pointer is fresh |

There is no published quota or SLA for the consumer update/item endpoints.
Bound global concurrency, honor `Retry-After`, cache negative outcomes, and
never evade upstream controls with proxy or IP rotation. Treat an endpoint
shape change or sustained 403/429 response as a circuit-breaker event, not a
reason to imitate Chrome more aggressively.

## Launch acceptance

The acquisition slice is ready to implement when all of these are true:

- FX Inline publisher authorization is recorded.
- The arbitrary-ID path is disabled unless broader permission is documented.
- Unit tests cover every accepted/rejected URL shape and SSRF lookalike.
- Fixture tests cover update XML status/error variants and secure parsing.
- Integration tests prove preflight rejection and streaming abort at byte
  30,000,001.
- A known FX Inline fixture proves hash, CRX3 publisher proof, derived ID,
  manifest version, immutable cache key, single-flight behavior, and cleanup.
- Simulated publication races, redirects, 429, 5xx, missing length, wrong
  length, wrong hash, same-version/different-hash, private/unavailable, and
  partial-rollout ambiguity all produce the typed outcomes above.
- Monitoring can disable acquisition without taking down the static portfolio
  and precomputed FX Inline report.

## Unresolved risks

1. Written Google permission for a third-party server to use the consumer
   update/item interfaces has not been found.
2. Each publisher may substitute an EULA; permission must be checked and
   retained per allowlisted extension.
3. Consumer endpoints have no documented stability or quota commitment.
4. Partial and geographic rollouts prevent an absolute “global current
   version” claim.
5. The implementation still needs a maintained CRX3 production-publisher-proof
   verifier; parsing a CRX envelope as ZIP is insufficient.
6. Legal review remains appropriate before enabling arbitrary third-party
   analysis or publishing findings about third parties.
