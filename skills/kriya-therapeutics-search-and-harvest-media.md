---
name: Search Kriya Therapeutics content and harvest the media library
description: Use cross-content search to find Kriya Therapeutics material by term (AAV, geographic atrophy, KRIYA-586, FGF21, Series D, PreCheck), route on the result `subtype` to the right resolver, and page through the 304-item media library for logos, pipeline graphics and press imagery with an incremental modified_after harvest.
api: openapi/kriya-therapeutics-content-openapi.yml
base_url: https://kriyatherapeutics.com/wp-json
operations:
  - searchContent
  - getNewsItem
  - getPage
  - listMedia
  - getMediaItem
generated: '2026-08-04'
method: generated
---

# Search Kriya Therapeutics content and harvest media

## Part 1 — Search

`searchContent` is the fastest way in when you have a term rather than an id. It searches across
every registered public type at once — pages, posts and news — and returns lightweight projections.

```
GET /wp/v2/search?search=gene&per_page=20
```

58 objects were indexed on 2026-08-04. Useful terms for this provider: `AAV`, `gene therapy`,
`geographic atrophy`, `thyroid eye disease`, `KRIYA-586`, `FGF21`, `MASH`, `NASH`, `trigeminal
neuralgia`, `Type 1 diabetes`, `Series D`, `PreCheck`, `ASGCT`, `ARVO`, `Redpin`, `Tramontane`,
`Everads`, `GMP`.

Each result is a projection, **not** a full record:

```json
{
  "id": 1883,
  "title": "Kriya Announces Presentations at ASGCT 2026 ...",
  "url": "https://kriyatherapeutics.com/news/...",
  "type": "post",
  "subtype": "news",
  "_links": { "self": [ { "embeddable": true, "href": "..." } ] }
}
```

### Route on `subtype`, not `type`

`type` is the search-index class and reads `post` for everything. **`subtype` tells you which
collection to resolve against:**

- `subtype: news` → `getNewsItem` (`GET /wp/v2/news/{id}`)
- `subtype: page` → `getPage` (`GET /wp/v2/pages/{id}`)
- `subtype: post` → `getPost` (`GET /wp/v2/posts/{id}`) — remember these are people records

You can pre-filter instead of routing after the fact:

```
GET /wp/v2/search?search=manufacturing&subtype=news&per_page=50
```

Or just follow `_links.self[0].href`, which is the same resource and avoids hard-coding routes.

Read `X-WP-Total` / `X-WP-TotalPages` for the result count; page with `page`/`per_page`.

## Part 2 — Harvest the media library

304 attachments on 2026-08-04 — the largest addressable collection on this API. Logos
(`Kriya-Logo-Bl.png`, `Kriya-Logo-Wh.png`, the `Kriya_KMark` favicon set), pipeline and laboratory
photography, leadership headshots, and the imagery attached to each press release.

```
GET /wp/v2/media?per_page=100&page=1&orderby=date&order=desc
```

Each item carries what you need without a second call:

- `source_url` — the full-size file
- `media_type` / `mime_type` — filter with `?media_type=image` or `?mime_type=image/png`
- `alt_text` — often descriptive on this site (e.g. "Kriya is a leading gene therapy company")
- `filesize`
- `media_details` — width, height, and **every generated size** (thumbnail, medium, large, plus
  theme sizes) with its own `source_url`. Pick the size you need instead of downloading the
  original.
- `post` — the id of the object the attachment is attached to (nullable). Resolve with `getPage` or
  `getNewsItem` depending on what the id turns out to be; ids are a shared sequence, so check.

### Incremental harvest

```
GET /wp/v2/media?modified_after=2026-05-01T00:00:00&per_page=100
```

Most recent attachment observed at harvest: id 1898, 2026-05-29. Store the highest `modified` you
have seen and window forward.

### Get one item

`getMediaItem` (`GET /wp/v2/media/{id}`) — use this to resolve a `featured_media` integer from a
news item, page or person record.

## Errors

| Status | Body | What to do |
|---|---|---|
| 400 | `rest_invalid_param` JSON | `per_page` must be 1–100 |
| 404 | **nginx HTML**, not JSON | The id from search no longer resolves, or you resolved against the wrong collection. Branch on `Content-Type`, then re-run the search |

## Etiquette

Search and media are unauthenticated and uncounted, but no rate-limit policy is published and no
rate-limit headers are returned. robots.txt asks for `Crawl-delay: 10`. Batch your terms, respect
the `max-age=600` cache window, and pull binaries once — `source_url` files are served from the same
Cloudflare edge and do not change.
