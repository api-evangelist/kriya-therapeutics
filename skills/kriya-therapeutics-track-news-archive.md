---
name: Track the Kriya Therapeutics press-release archive
description: Page, filter and incrementally harvest the 28-item `news` custom post type — Kriya Therapeutics' full financing, acquisition, licensing, appointment and scientific-presentation history from May 2020 to June 2026 — and parse the Gutenberg block HTML in each body.
api: openapi/kriya-therapeutics-content-openapi.yml
base_url: https://kriyatherapeutics.com/wp-json
operations:
  - listNews
  - getNewsItem
  - listNewsCategories
  - listTypes
generated: '2026-08-04'
method: generated
---

# Track the Kriya Therapeutics press-release archive

This is the highest-value thing on this API. Most corporate WordPress sites in the catalog bury
their press releases inside one hand-authored page; Kriya Therapeutics registered a `news` custom
post type, so every release is an addressable, dated, filterable record with clean HTML in the body.

## Steps

### 1. Confirm the collection is still there

Custom post types are a plugin/theme decision and can disappear on a redeploy. Call `listTypes`
(`GET /wp/v2/types`) and check for a `news` key with `rest_base: news`. Do not hard-code the route
without this check if you are running unattended.

### 2. List

```
GET /wp/v2/news?per_page=100&orderby=date&order=desc
```

`per_page` maxes at 100 (101+ returns `400 rest_invalid_param`), and the collection held 28 items on
2026-08-04, so one request covers the whole archive. Read `X-WP-Total` to confirm the count and
`X-WP-TotalPages` before assuming you got everything.

Cut the payload down with `_fields` when you only need the index:

```
GET /wp/v2/news?per_page=100&_fields=id,date,slug,title,link
```

### 3. Harvest incrementally

For a recurring job, use the date windows rather than re-pulling everything:

- `modified_after=<ISO8601>` — catches edits to existing releases as well as new ones. This is the
  one you want for a watcher.
- `after=<ISO8601>` / `before=<ISO8601>` — filter on publication date, for building a window
  (e.g. everything since the last financing round).

```
GET /wp/v2/news?modified_after=2026-01-01T00:00:00&per_page=100
```

### 4. Read a release

Call `getNewsItem` (`GET /wp/v2/news/{id}`) or take `content.rendered` straight off the list
response. Bodies are **Gutenberg block markup** — semantic `<p class="wp-block-paragraph">`, `<a>`,
`<em>`, `<figure class="wp-block-image">` — not page-builder shortcodes. So:

- Strip tags, or keep them; either works. No shortcode cleanup is required.
- **Decode HTML entities.** The bodies use `&#8211;`, `&#8217;` and `’` heavily; a raw string
  comparison against a press-release title will fail without decoding.
- `excerpt.rendered` gives you a short form when you do not want the whole release.

### 5. Do not expect taxonomy or authors to help

- `newscategories` is declared on every item and is **always an empty array** — the taxonomy is
  registered with zero terms. `listNewsCategories` returns `[]`. Filtering by it returns nothing.
- `author` is an integer that **cannot be resolved**: `/wp/v2/users` returns `401
  rest_user_cannot_view` on this deployment. Treat it as opaque; never present it as a byline.
- `featured_media` IS resolvable via `getMediaItem`.

## What is in the archive

28 items spanning 2020-05-12 to 2026-06-29: every financing round (Series A $80M, Series B $100M,
Series C $270M + $150M extension, Series D $320M), the Redpin Therapeutics (2022) and Tramontane
Therapeutics (2023) acquisitions, the Everads license (2023), the C-suite appointments (COO, CLO,
Chief Gene Therapy Officer, CMO, CFO), the preclinical data disclosures (KRIYA-586, AAV-FGF21) and
the conference presentations (ASGCT, ARVO, J.P. Morgan). This is the company's own dated record of
itself — treat it as the primary source, not the ZoomInfo/Crunchbase mirrors.

## Errors

| Status | Body | What to do |
|---|---|---|
| 400 | `{"code":"rest_invalid_param",...}` JSON | `per_page` must be 1–100; read `data.params` |
| 404 | **nginx HTML**, not JSON | An unknown id is intercepted by the edge. Branch on `Content-Type` before parsing or your client will throw a JSON parse error instead of seeing a 404 |

## Etiquette

Unauthenticated and uncounted, but no rate-limit policy is published and no rate-limit headers are
returned. The site's robots.txt asks for `Crawl-delay: 10`. Responses carry `Cache-Control:
max-age=600, must-revalidate` and no ETag, so there is no conditional GET — poll no more than once
per ten minutes, and prefer `modified_after` over re-pulling.
