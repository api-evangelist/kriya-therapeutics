---
name: Read Kriya Therapeutics pipeline and platform content
description: Retrieve the 18 published pages — Pipeline and its three therapeutic-area children, Product Design, Research & Development, Platform/manufacturing, Team, Careers, Connect and the legal pages — plus the people records that back the Team page, which live in `posts` categorised into leadership/bod/sab rather than in the empty `team` collection.
api: openapi/kriya-therapeutics-content-openapi.yml
base_url: https://kriyatherapeutics.com/wp-json
operations:
  - listPages
  - getPage
  - listPosts
  - getPost
  - listCategories
  - listTeam
  - listTypes
generated: '2026-08-04'
method: generated
---

# Read Kriya Therapeutics pipeline and platform content

The scientific narrative — what the company is developing, how it designs products, and what it can
manufacture — lives in the `pages` collection. The people live somewhere non-obvious.

## Steps

### 1. Index the pages

```
GET /wp/v2/pages?per_page=100&_fields=id,slug,title,link,parent,menu_order
```

18 published pages on 2026-08-04. The ones that carry substance:

| id | slug | What it is |
|---|---|---|
| 23 | `pipeline` | The full pipeline across three therapeutic areas |
| 1265 | `ophthalmology` | Geographic atrophy, thyroid eye disease (KRIYA-586) |
| 1290 | `metabolic-disease` | Type 1 diabetes, MASH/NASH |
| 1282 | `neurology` | Trigeminal neuralgia |
| 1627 | `product-design` | Computational + experimental product design |
| 1628 | `research-development` | Discovery and translational research |
| 1614 | `manufacturing` | Titled "Platform" — in-house GMP, 1L→3,000L |
| 1481 | `our-team` | Titled "Team" |
| 1555 | `newsroom` | Titled "News" — the human view of the news collection |
| 955 | `careers` | Open roles |
| 15 | `connect` | Contact |

Ignore id 1764 (`styles`) — it is a theme utility page, not content.

### 2. Read a page

`getPage` (`GET /wp/v2/pages/{id}`), or filter the list by slug:

```
GET /wp/v2/pages?slug=pipeline
```

`content.rendered` is **Gutenberg block markup**: semantic HTML with `wp-block-*` classes, plus a
few theme blocks (`wp-block-kriya-blocks-page-hero`, `bit_page_hero`, `bit-grid`). No shortcode
stripping is needed, but there is a lot of presentational wrapper markup and inline `srcset` — reduce
to text and drop the layout `div`s before feeding it to anything.

### 3. Get the people — NOT from `listTeam`

This is the trap on this deployment. A `team` custom post type **is** registered, with its own
`teamkeywords` taxonomy carrying four terms (Leadership, Board of Directors, Founders, Scientific and
Strategic Advisors). `listTeam` (`GET /wp/v2/team`) returns `[]` — `X-WP-Total: 0` — and every
`teamkeywords` term reports `count: 0`.

The people records are core **posts**, categorised with the `category` taxonomy:

```
GET /wp/v2/categories?per_page=100
```

| id | slug | count |
|---|---|---|
| 20 | `leadership` | 17 |
| 21 | `bod` | 10 |
| 22 | `sab` | 6 |

So:

```
GET /wp/v2/posts?categories=20&per_page=100        # leadership
GET /wp/v2/posts?categories=21&per_page=100        # board of directors
GET /wp/v2/posts?categories=22&per_page=100        # scientific/strategic advisors
```

33 posts total. Each carries the person's name in `title.rendered`, their bio in `content.rendered`,
and a headshot via `featured_media` → `getMediaItem`. The `category` taxonomy on this site encodes
**people groupings, not topics** — do not read it as an editorial taxonomy.

### 4. Verify before you rely on it

`listTypes` (`GET /wp/v2/types`) tells you whether `team` has been populated since this skill was
written. If `listTeam` ever starts returning items, prefer it over the category workaround.

## Errors

| Status | Body | What to do |
|---|---|---|
| 400 | `rest_invalid_param` JSON | `per_page` 1–100 |
| 404 | **nginx HTML**, not JSON | Unknown page id — branch on `Content-Type` before parsing |
| 401 | `rest_user_cannot_view` | You tried `/wp/v2/users`. Author ids are not resolvable here |

## Etiquette

Read-only, unauthenticated. `Cache-Control: max-age=600`, no ETag. Cache the page index; the pages
change rarely (most recent `page-sitemap.xml` lastmod was 2026-07-09).
