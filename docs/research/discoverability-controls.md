# Discoverability controls for the DocX showcase

Research date: 2026-07-30  
Decision ticket: [Define search and AI-crawler discoverability controls](https://github.com/SpamArtist/docx-showcase/issues/4)

## Decision

Publish one canonical, indexable document: the recruiter-facing page at
`PUBLIC_ORIGIN/`. Return its complete portfolio content as semantic HTML in the
initial `200 OK` response, then hydrate only the scanner and progress UI.

Treat live reports as transient application state, not web pages:

- never create a report URL, route, query parameter, fragment, share action, or
  sitemap entry;
- send extension submissions with `POST /api/scans`;
- return job state and the sanitized report only to a short-lived bearer
  capability held in JavaScript memory, never in a URL or browser storage;
- apply `X-Robots-Tag: noindex, nofollow, nosnippet, noarchive` and
  `Cache-Control: private, no-store` to every job/report/API response;
- reject requests without a valid capability before returning job or report
  content; and
- reserve and crawl-block `/api/`, `/jobs/`, `/reports/`, and `/_internal/`.

The main page is deliberately public and sanitized, so the default
`User-agent: *` group should allow all compliant automatic crawlers to crawl it.
This includes search, AI-search, and model-development crawlers. The server must
still enforce the operational boundary because the Robots Exclusion Protocol
is not authorization, and some user-triggered AI fetchers say that robots rules
may not apply.

This policy maximizes the project's only goal—helping X get hired—without
making third-party reports into publishable resources.

## What is a standard, and what is not

| Control | Status | Role in this project |
| --- | --- | --- |
| `/robots.txt` `User-agent`, `Allow`, and `Disallow` | IETF Standards Track, [RFC 9309](https://www.rfc-editor.org/rfc/rfc9309.html) | Crawl preference, never access control |
| HTTP caching directives | Internet Standard, [RFC 9111](https://www.rfc-editor.org/rfc/rfc9111.html); `immutable` is Standards Track [RFC 8246](https://www.rfc-editor.org/rfc/rfc8246.html) | Fresh public HTML/assets; prevent HTTP storage of transient responses |
| `rel="canonical"` | Registered link relation, [RFC 6596](https://www.rfc-editor.org/rfc/rfc6596.html), with current [Google implementation guidance](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls) | Consolidate all public-page variants to one HTTPS URL |
| XML Sitemap | Cross-engine protocol published by [sitemaps.org](https://www.sitemaps.org/protocol.html), supported by [Google](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap) and [Bing](https://www.bing.com/webmasters/help/Sitemaps-3b5cf6ed) | Advertise only the canonical public page |
| Robots `meta` and `X-Robots-Tag` | Vendor-supported convention, not part of RFC 9309; current [Google](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag) and [Bing](https://www.bing.com/webmasters/help/robots-meta-tags-and-attributes-that-bing-supports-5198d240) documentation | Index/preview the main page; add defense in depth to operational responses |
| Schema.org JSON-LD | Shared vocabulary/convention; Google recommends JSON-LD and requires markup to describe visible page content in its [structured-data guidance](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data) | Give machines an accurate entity graph, not a ranking promise |
| Open Graph metadata | Optional [Open Graph protocol](https://ogp.me/), not a search-indexing standard | Deterministic social/link previews for the public page |
| Text alternatives | WCAG 2.2 Level A requirement; [SC 1.1.1](https://www.w3.org/WAI/WCAG22/Understanding/non-text-content.html) | Make diagrams understandable to people and non-visual consumers |
| `/llms.txt` | An informal, community-maintained [proposal](https://llmstxt.org/), not an IETF/W3C standard and not a substitute for robots or sitemaps | Cheap optional orientation file after the standards-based launch contract is green |

Neither crawling nor a sitemap guarantees indexing. Google calls sitemap
submission a hint, while Bing says sitemaps improve discovery but do not
guarantee visibility. Search Console and Bing Webmaster Tools remain the
verification surfaces.

## Public main-page contract

### URL and response

`PUBLIC_ORIGIN` is the one deploy-time HTTPS origin selected by the hosting
decision. The canonical URL is exactly:

```text
${PUBLIC_ORIGIN}/
```

Deployment must fail while `PUBLIC_ORIGIN` is unset or contains a path. HTTP,
alternate hostnames, `/index.html`, and query-bearing variants must issue a
permanent redirect to that URL. Section fragments may navigate within the page,
but the canonical remains the fragment-free root. The scanner must never put
the submitted Chrome Web Store URL, extension ID, version, job ID, or report
data in `location`, browser history, or a referrer.

The root response must be:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Language: en
Cache-Control: public, max-age=0, must-revalidate
ETag: "<content-derived-validator>"
```

`no-cache`/`max-age=0` permits storage but requires validation before reuse;
`no-store` forbids storage. That distinction is why public HTML is revalidated
while transient report delivery is not stored. RFC 9111 also warns that
`no-store` alone is not a privacy mechanism, so the authorization and routing
controls below are mandatory.

### Rendering

The initial HTML—not a JavaScript-created app shell—must contain:

- the hero and “Hire X” GitHub link;
- a concise explanation of DocX and how it works;
- the version-labelled, curated FX Inline sample report;
- reproducible performance measurements and benchmark context;
- the “How This Was Built” narrative;
- the visible text equivalents of architecture/process diagrams;
- curated public code-snippet explanations; and
- the live scanner form and its size/rate-limit explanation.

The scanner, polling, progress, and in-session report panel may hydrate on the
client. The portfolio narrative may not depend on hydration. Google can render
JavaScript, but says server-side or pre-rendering remains beneficial because
not all bots execute JavaScript; its processing model can also delay rendering.
See [Google's JavaScript SEO guidance](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics).

Use landmark elements (`header`, `main`, `nav`, `section`, `footer`), one
descriptive `h1`, a logical heading hierarchy, and real `<a href>` links.
Google documents that ordinary anchor elements with `href` are the reliable
crawlable link shape in its [link guidance](https://developers.google.com/search/docs/crawling-indexing/links-crawlable).

### HTML metadata

Render the following directly in the original `<head>`:

```html
<title>DocX — Evidence-led Chrome extension analysis by X</title>
<meta
  name="description"
  content="See a curated DocX report for FX Inline, reproducible performance evidence, and the public process behind the work."
/>
<meta
  name="robots"
  content="index, follow, max-snippet:-1, max-image-preview:large"
/>
<link rel="canonical" href="${PUBLIC_ORIGIN}/" />
```

The absolute self-referencing canonical must agree with redirects, internal
links, the sitemap, `og:url`, and JSON-LD URLs. Google recommends putting a
self-referencing canonical in the source HTML and not changing it with
JavaScript.

Do not use `noarchive`, `nocache`, or `nosnippet` on the main page. In
particular, Bing currently assigns generative-AI and Copilot effects to
`noarchive` and `nocache`, and Google's `nosnippet` also prevents direct use as
input to AI Overviews and AI Mode. Those directives conflict with the
visibility objective.

### Structured data

Emit one server-rendered JSON-LD graph. Replace template variables at build
time, and include only facts also visible on the page:

```json
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "WebSite",
      "@id": "${PUBLIC_ORIGIN}/#website",
      "url": "${PUBLIC_ORIGIN}/",
      "name": "DocX",
      "description": "Evidence-led Chrome extension analysis and a public engineering showcase.",
      "inLanguage": "en"
    },
    {
      "@type": "WebApplication",
      "@id": "${PUBLIC_ORIGIN}/#application",
      "url": "${PUBLIC_ORIGIN}/",
      "name": "DocX",
      "description": "A curated, recruiter-facing Chrome extension analysis experience.",
      "applicationCategory": "DeveloperApplication",
      "operatingSystem": "Web",
      "isAccessibleForFree": true,
      "author": {
        "@id": "${PUBLIC_ORIGIN}/#x"
      }
    },
    {
      "@type": "Person",
      "@id": "${PUBLIC_ORIGIN}/#x",
      "name": "X",
      "sameAs": "https://github.com/SpamArtist"
    }
  ]
}
```

Do not mark up the FX Inline sample as a review, claim ownership of it, invent
ratings, or add `JobPosting`. Validate the deployed graph with Schema.org's
validator and Google's Rich Results Test. Structured data improves machine
understanding; it does not guarantee a rich result.

### Social metadata

The public page gets one versioned, static preview image containing no
third-party report details beyond the already-public FX Inline sample summary.
Add the Open Graph protocol's four base fields plus accessibility/context
fields:

```html
<meta property="og:type" content="website" />
<meta property="og:title" content="DocX — Evidence-led Chrome extension analysis by X" />
<meta property="og:description" content="A recruiter-focused DocX showcase with a live scanner, reproducible benchmarks, and a public build record." />
<meta property="og:url" content="${PUBLIC_ORIGIN}/" />
<meta property="og:site_name" content="DocX" />
<meta property="og:locale" content="en_US" />
<meta property="og:image" content="${PUBLIC_ORIGIN}/assets/docx-showcase-og.<hash>.png" />
<meta property="og:image:width" content="1200" />
<meta property="og:image:height" content="630" />
<meta property="og:image:alt" content="DocX Chrome extension analysis showcase by X" />
```

The protocol defines `og:title`, `og:type`, `og:image`, and `og:url` as its
required base properties and recommends `og:description` and `og:site_name`;
it also specifies `og:image:alt` when an image is present. The 1200×630
dimensions are a project convention, not part of that protocol.

### Diagrams and text fallbacks

Every architecture or process diagram must have:

1. a short accessible name;
2. a visible adjacent prose summary of the conclusion or flow; and
3. a full ordered-list, definition-list, or table representation of all
   meaningful nodes and edges.

Use `<figure>`/`<figcaption>` and associate any longer text with the image/SVG
using `aria-describedby`. If Mermaid is used, pre-render it at build time; do
not make the Mermaid client runtime the only source of meaning. W3C's
[complex-image guidance](https://www.w3.org/WAI/tutorials/images/complex/)
recommends a brief alternative plus a nearby or linked long description for
charts and diagrams. This is both the accessible fallback and the
machine-readable account—do not add hidden SEO-only prose.

## `robots.txt`

Serve UTF-8 `text/plain` at the exact root path `/robots.txt`, as RFC 9309
requires:

```text
User-agent: *
Allow: /
Disallow: /api/
Disallow: /jobs/
Disallow: /reports/
Disallow: /_internal/

Sitemap: ${PUBLIC_ORIGIN}/sitemap.xml
```

Do not add crawler-specific groups unless a later policy intentionally differs.
Under RFC 9309, a crawler uses its matching specific group instead of the
wildcard group; adding an incomplete bot-specific group can therefore
accidentally remove the operational disallows. Bing also warns that specific
groups need their general rules repeated.

The wildcard policy intentionally lets future and currently named automatic
bots reach only the public surface, including model-development bots. The
public projection must therefore remain safe to redistribute. `robots.txt` is
public, exposes the existence of blocked path prefixes, and is explicitly not
an authorization mechanism in RFC 9309.

## Sitemap

Serve `/sitemap.xml` as UTF-8 XML and list only the canonical root:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>${PUBLIC_ORIGIN}/</loc>
    <lastmod>YYYY-MM-DD</lastmod>
  </url>
</urlset>
```

`lastmod` must be the date the meaningful public content last changed, not the
deployment or sitemap-generation time. Do not list API routes, report/job
routes, assets, `robots.txt`, or `llms.txt`. Reference the sitemap in
`robots.txt`, then submit and monitor it in both Google Search Console and Bing
Webmaster Tools.

## AI-crawler policy

The wildcard policy produces the following explicit outcome:

| Vendor/product | Official token | Public root | Operational prefixes | Important vendor behavior |
| --- | --- | --- | --- | --- |
| Google Search | `Googlebot` | Allow | Deny | Search indexing; do not block render-critical public CSS/JS |
| Microsoft Bing/Copilot | `bingbot` | Allow | Deny | Keep `noarchive`, `nocache`, and `nosnippet` off the root to avoid restricting Copilot/search use |
| OpenAI search | `OAI-SearchBot` | Allow | Deny | Controls appearance in ChatGPT search independently from GPTBot |
| OpenAI model development | `GPTBot` | Allow | Deny | Public, sanitized root may be used for model development |
| OpenAI user fetch | `ChatGPT-User` | Allow at server | Require capability; otherwise `401`/`404` | OpenAI says robots rules may not apply to user-initiated actions |
| Anthropic search | `Claude-SearchBot` | Allow | Deny | Supports Claude search-result quality |
| Anthropic model development | `ClaudeBot` | Allow | Deny | Public, sanitized root may contribute to future model training |
| Anthropic user fetch | `Claude-User` | Allow at server | Require capability; otherwise `401`/`404` | Anthropic documents it separately from automatic/search crawling |
| Google Gemini control | `Google-Extended` | Allow | Deny | Google currently couples future Gemini training and Gemini grounding under this token; it does not affect Google Search |
| Apple search/grounding | `Applebot` | Allow | Deny | Leave `nosnippet` off the root so Apple may use current content in generated answers |
| Apple model development | `Applebot-Extended` | Allow | Deny | Public, sanitized root may contribute to Apple foundation-model training |
| Perplexity search | `PerplexityBot` | Allow | Deny | Perplexity says this bot is for search, not foundation-model training |
| Perplexity user fetch | `Perplexity-User` | Allow at server | Require capability; otherwise `401`/`404` | Perplexity says this fetcher generally ignores robots rules |
| Unknown compliant crawler | wildcard | Allow | Deny | Future discovery is opt-out, but only for the intentionally public projection |

Sources for the named controls:

- [OpenAI crawler documentation](https://developers.openai.com/api/docs/bots)
  separates `OAI-SearchBot`, `GPTBot`, and user-triggered `ChatGPT-User`, and
  says each setting is independent.
- [Anthropic crawler documentation](https://support.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler)
  separates `ClaudeBot`, `Claude-SearchBot`, and `Claude-User`.
- [Google's common-crawler documentation](https://developers.google.com/crawling/docs/crawlers-fetchers/google-common-crawlers)
  says `Google-Extended` controls both specified Gemini training and grounding
  uses without affecting Google Search.
- [Applebot documentation](https://support.apple.com/en-us/119829) separates
  Apple search, generated-answer snippets, and `Applebot-Extended` training
  control.
- [Perplexity crawler documentation](https://docs.perplexity.ai/docs/resources/perplexity-crawlers)
  separates `PerplexityBot` from user-triggered `Perplexity-User` and publishes
  current IP ranges for verification.

### WAF conflict to avoid

Some vendors recommend permitting their published IP ranges through a WAF.
Never implement that as a site-wide bypass. A verified bot may bypass only
generic bot-challenge rules on `/`, `/assets/*`, `/robots.txt`,
`/sitemap.xml`, and optional `/llms.txt`. Bot identity must not bypass:

- bearer-capability checks;
- HTTP method restrictions;
- input validation;
- rate limits;
- the 30 MB download limit; or
- any `/api/`, `/jobs/`, `/reports/`, or `/_internal/` rule.

User-triggered AI fetchers create the sharpest conflict: allowing them to read
the public page improves recruiter discovery, but their vendors say they may
ignore robots rules. The only reliable boundary is to never expose a report as
an unauthenticated resource.

## Non-indexed, non-shareable operational contract

### Route and capability design

1. `POST /api/scans` accepts the Chrome Web Store URL and returns
   `{ jobId, capability, expiresAt }`.
2. `jobId` is an opaque random identifier; `capability` is an independent,
   high-entropy bearer secret.
3. The browser keeps both values in component/module memory only. Do not use
   query strings, fragments, cookies, `localStorage`, `sessionStorage`,
   IndexedDB, service-worker caches, or browser history.
4. `GET /api/jobs/{jobId}` requires
   `Authorization: Bearer <capability>`. It returns progress or the sanitized
   Portfolio Report in JSON.
5. The capability expires shortly after completion (recommended: 15 minutes)
   and is invalidated when the page is closed or reloaded from the client's
   perspective.
6. Reload returns the canonical portfolio page with the FX Inline sample.
   Resubmitting the same store URL obtains a new capability and reads the
   server-side extension-ID-plus-version cache without rerunning DocX.
7. No unauthenticated GET returns report existence, extension identity,
   progress, or report content. Missing/invalid capabilities receive `404`
   where existence itself should not be disclosed, otherwise `401`.

This makes a URL alone non-shareable. With anonymous, cookieless access, no
web design can stop a visitor from copying response content or deliberately
sharing a live bearer secret. The contract prevents accidental/stable sharing;
absolute recipient binding would require an account, cookie, device key, or
fragile IP binding and is outside the agreed privacy model.

### Operational response headers

Apply these headers to successes and errors under every reserved operational
prefix:

```http
Cache-Control: private, no-store
X-Robots-Tag: noindex, nofollow, nosnippet, noarchive
Referrer-Policy: no-referrer
X-Content-Type-Options: nosniff
```

JSON responses must use `Content-Type: application/json; charset=utf-8`.
Unused human-looking `/jobs/*` and `/reports/*` routes return real `404`
responses, not the SPA shell. `GET /api/scans` returns `405`.

The `robots.txt` disallow and `X-Robots-Tag` are intentionally redundant but
not equivalent. Google notes that a crawler cannot read `noindex` on a URL it
is forbidden to crawl. Here that is acceptable because the server never
returns public content at the URL: authorization plus `401`/`404`/`405` is the
primary control, `Disallow` prevents routine crawling, and the header protects
against accidental fetches by crawlers that do reach a response.

The persistent sanitized report cache is an origin-side application cache. It
does not authorize public delivery and must not cause the report response to
be cached by a browser or CDN.

## HTTP cache matrix

| Resource | Required `Cache-Control` | Validator/notes |
| --- | --- | --- |
| Canonical HTML `/` | `public, max-age=0, must-revalidate` | Strong ETag generated from content |
| Hashed CSS, JS, images, fonts | `public, max-age=31536000, immutable` | Filename changes with bytes; immutable semantics come from RFC 8246 |
| `/robots.txt`, `/sitemap.xml`, optional `/llms.txt` | `public, max-age=3600` | ETag; keep propagation delay bounded |
| `POST /api/scans` and all job/report/API responses | `private, no-store` | Never CDN-cache; no service-worker cache |
| Operational `401`, `403`, `404`, `405`, `429`, `5xx` | `private, no-store` | Prevent negative/status leakage and stale limits |

## Optional `llms.txt`

After the standards-based page, sitemap, and crawler verification pass,
publishing a concise `/llms.txt` is a low-cost enhancement. It should contain:

```markdown
# DocX Showcase

> A recruiter-facing demonstration of DocX, Chrome extension analysis work by X.

The live scanner returns private, in-session reports. Do not infer or request
undocumented report or API URLs.

## Public portfolio

- [DocX showcase](${PUBLIC_ORIGIN}/): Product explanation, FX Inline sample, benchmarks, process diagrams, and hiring link.
- [Public build record](<PUBLIC_PROJECT_URL>): Sanitized decisions and implementation record.
```

Do not list operational routes or reproduce private report data. Do not delay
launch for this file, claim that major crawlers honor it, place it in the
sitemap, or treat it as consent/access control. Its own specification calls it
a proposal intended to coexist with robots and sitemaps.

## Acceptance checks

Automate these checks against preview and production:

1. Fetch `/` without executing JavaScript and assert `200`, the final title,
   one `h1`, every portfolio section, the FX Inline version, diagram text, the
   canonical, robots meta, Open Graph tags, and JSON-LD.
2. Fetch every alternate origin/path/query form and assert its permanent
   redirect ends at exactly `PUBLIC_ORIGIN/`.
3. Fetch `/robots.txt` with Googlebot, bingbot, OAI-SearchBot, GPTBot,
   Claude-SearchBot, ClaudeBot, Google-Extended, Applebot,
   Applebot-Extended, and PerplexityBot user agents; validate that `/` is
   allowed and all four operational prefixes are denied.
4. Assert `/sitemap.xml` is valid XML and its only `<loc>` is the canonical
   root with an accurate content `lastmod`.
5. Exercise every operational success/error response and assert
   `Cache-Control: private, no-store` and the complete `X-Robots-Tag`.
6. Assert a copied `jobId` without its bearer capability cannot read status or
   report data; assert the token never appears in URL, logs intended for
   analytics, storage APIs, or the `Referer` header.
7. Disable JavaScript and verify the entire recruiter narrative and text
   alternatives remain readable; only live scanning may be unavailable.
8. Run the deployed root through Schema.org validation, Google's Rich Results
   Test and URL Inspection, and Bing Webmaster Tools URL Inspection. Submit
   the one-URL sitemap to Google and Bing.
9. Inspect CDN/WAF configuration to ensure bot allowances are public-path
   scoped and cannot bypass operational controls.
10. Confirm hashed assets get the one-year immutable policy and the HTML,
    crawler files, and operational surfaces get their respective cache
    policies.

## Final allow/deny matrix

| Surface | Human visitor | Search bots | AI search/retrieval bots | Model-development bots | Sitemap | Index directive | HTTP cache | Shareability |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Canonical `/` | Allow | Allow | Allow | Allow | Include | `index, follow`; previews allowed | Public, revalidate | Normal public URL |
| Hashed public assets | Allow | Allow | Allow | Allow | Exclude | Inherit/reference use | Public, 1 year, immutable | Public |
| `/robots.txt` | Allow | Allow | Allow | Allow | Exclude | Not applicable | Public, 1 hour | Public |
| `/sitemap.xml` | Allow | Allow | Allow | Allow | Self excluded | Not applicable | Public, 1 hour | Public |
| Optional `/llms.txt` | Allow | Allow | Allow | Allow | Exclude | No special claim | Public, 1 hour | Public |
| `POST /api/scans` | Rate-limited allow | Deny crawl | Deny crawl | Deny crawl | Exclude | `X-Robots-Tag: noindex...` | Private, no-store | No GET/share URL |
| `/api/jobs/*` | Valid bearer only | Deny | Deny; valid user fetch still needs bearer | Deny | Exclude | `X-Robots-Tag: noindex...` | Private, no-store | ID alone reveals nothing |
| `/jobs/*`, `/reports/*` | `404` | Deny | Deny | Deny | Exclude | `X-Robots-Tag: noindex...` | Private, no-store | No report pages exist |
| `/_internal/*` | Deny | Deny | Deny | Deny | Exclude | `X-Robots-Tag: noindex...` | Private, no-store | Never public |

## Residual risks

- Broad root access permits model-development crawlers to use the intentionally
  public, sanitized showcase. If X later wants search visibility without
  training, add complete crawler-specific groups; note that Google currently
  couples Gemini grounding and specified training uses under
  `Google-Extended`.
- Vendors can change crawler tokens, products, IP ranges, and directive
  semantics. Review the linked official pages before launch and quarterly
  thereafter.
- Compliant crawler directives cannot prevent malicious scraping, screenshots,
  copied JSON, or a visitor sharing a live bearer secret. The only sound
  secrecy boundary is omitting sensitive fields from the Portfolio Report.
- A crawler can retain an old `robots.txt` briefly. RFC 9309 permits caching,
  generally no longer than 24 hours when reachable; OpenAI and Perplexity each
  warn that changes may take about 24 hours to propagate.
- The final hostname is deliberately deferred to the hosting decision.
  Production acceptance must replace `PUBLIC_ORIGIN` everywhere and prove all
  canonical signals agree before indexing is enabled.
