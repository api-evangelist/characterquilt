---
name: characterquilt-brand-profile-lookup
description: >-
  Look up a company's extracted brand identity — primary/secondary/accent colours,
  typography and font stacks, logo and favicon URLs, button component styles, brand
  personality and design-system signals — from CharacterQuilt's public brand-profiles
  data surface. Use when you need on-brand colours or fonts for a named company, when
  matching creative to a company's visual identity, or when checking whether a company
  is catalogued at all.
api: CharacterQuilt Brand Profiles
operations:
  - getLlmsIndex
  - getBrandProfile
  - getBrandProfilePage
generated: '2026-08-13'
method: generated
source: >-
  openapi/characterquilt-discovery-api-openapi.yml,
  openapi/characterquilt-branding-api-openapi.yml,
  conventions/characterquilt-conventions.yml,
  errors/characterquilt-problem-types.yml
---

# CharacterQuilt brand profile lookup

Read-only. No credential of any kind — this surface is unauthenticated and
CORS-open (`access-control-allow-origin: *`). Base host: `https://www.characterquilt.com`.

> Authored by API Evangelist from CharacterQuilt's observed public surface.
> CharacterQuilt publishes no API reference; every operation below is grounded in a
> real operationId in `openapi/` and was verified live on 2026-08-13.

## Step 1 — resolve the company to a slug (`getLlmsIndex`)

```
GET https://www.characterquilt.com/llms.txt
```

Returns `text/plain`, ~350 KB, ~6,250 lines. There is **no** search endpoint and
**no** paginated index — this one document is the entire catalog. Under
`## Brand profiles` each line is:

```
- [Company Name](/branding/company-slug): #PRIMARYHEX · Primary Font
```

Parse that line and you already have the slug, the primary brand colour and the
primary font — often enough to skip step 2 entirely.

Slugs are lowercase and hyphenated (`scale-ai`, `1password`, `zo-computer`).
Do not guess a slug from a company name; resolve it here. Cache the document and
revalidate with `If-None-Match` rather than refetching 350 KB.

## Step 2 — fetch the full profile (`getBrandProfile`)

```
GET https://www.characterquilt.com/branding/{slug}.json
```

`{slug}` is required and comes from step 1. Returns `application/json`
(~26 KB for a well-populated company). The `BrandProfile` shape:

| Field | What it holds |
|---|---|
| `name`, `slug`, `domain` | Company identity |
| `sources` | Where the company was sourced from (e.g. `yc`, `forbes_cloud_100`) |
| `branding.colors` | `primary`, `secondary`, `accent`, `background`, `textPrimary`, `link` |
| `branding.colorScheme` | `light` or `dark` |
| `branding.fonts` | Array of `{family, role}` |
| `branding.typography` | `fontFamilies`, `fontStacks`, `fontSizes` |
| `branding.spacing` | `baseUnit`, `borderRadius` |
| `branding.components` | `buttonPrimary`, `buttonSecondary` style objects |
| `branding.images` | `logo`, `favicon`, `ogImage`, `logoHref` |
| `branding.personality` | `tone`, `energy`, `targetAudience` |
| `branding.designSystem` | `framework`, `componentLibrary` |
| `branding.confidence` | `buttons`, `colors`, `overall` — **read this before trusting the values** |
| `images[]` | Harvested assets with `image_url`, `width`, `height`, `bytes` |
| `font_resources[]` | `family` + `google_fonts_url` |
| `agent_instructions` | Provider-supplied guidance for agents consuming the profile |

Always check `branding.confidence.overall` before applying a colour or font
downstream. The profiles are machine-extracted from live websites; a low
confidence score means the extraction is a guess.

## Step 3 — the human view, when you need it (`getBrandProfilePage`)

```
GET https://www.characterquilt.com/branding/{slug}
```

Same slug, no `.json`, returns `text/html`. Format is selected by **path
extension, not by the `Accept` header** — asking for JSON via `Accept` on the
extensionless path will still return HTML.

## Errors

| Status | Body | Meaning |
|---|---|---|
| 404 | `The page could not be found  NOT_FOUND  <trace-id>` (plain text) | The slug is not catalogued |

The 404 body is **plain text, not JSON** — do not attempt to parse it. There is no
RFC 9457 problem+json envelope on this surface. A 404 does not distinguish "never
catalogued" from "withdrawn"; re-resolve against `/llms.txt` before concluding a
company is absent.

## Runtime rules

- **No rate limits are published and no rate-limit headers are returned.** There is
  no `X-RateLimit-*`, no `RateLimit-*` and no `Retry-After` on any response, so you
  cannot back off proactively. Be conservative: fetch `/llms.txt` once per session,
  cache profiles, and parallelise gently.
- **Use conditional requests.** Responses carry `etag` and `last-modified` with
  `cache-control: public, max-age=0, must-revalidate`. This is the only mechanism
  available for avoiding repeat transfer.
- **No versioning.** Nothing in the path, headers or body identifies a version, so
  the response shape can change without notice. Validate fields defensively rather
  than assuming presence.
- **No idempotency contract** — not needed here, every operation is a read.
- **Query strings are discouraged**: `robots.txt` disallows `/*?*`. No operation on
  this surface takes a query parameter.

## What this skill does NOT cover

CharacterQuilt's actual product — the marketing agents — sits behind a separate,
OAuth-protected MCP server at `https://mcp.characterquilt.com/api/mcp`
(scopes: `read:design_brain`, `write:generated_artifacts`, `publish:public_file`,
`read:agent_work`, `write:agent_work`). That surface is undocumented and its tool
list is auth-gated. See `mcp/characterquilt-mcp.yml`. Nothing in this skill will
reach it.
