---
name: sinclair-press-release-monitoring
description: >-
  Search, page and monitor Sinclair, Inc.'s press releases and newsroom items over the
  corporate WordPress REST API. Use when asked what Sinclair has announced recently,
  to find announcements on a topic such as ATSC 3.0 or NextGen TV, to track executive
  appointments and station launches, or to build a change feed over Sinclair corporate
  communications.
api: Sinclair Corporate Content API
base_url: https://sbgi.net/wp-json
operations:
  - listPosts
  - getPost
  - search
  - listCategories
  - getLatestPosts
generated: '2026-08-12'
method: generated
source: openapi/sinclair-broadcast-group-content-openapi.yml
---

# Monitoring Sinclair press releases

1,753 press releases are readable anonymously. This is the richest queryable corpus on
the surface, and unlike the RSS feed it supports date ranges, category filters and
full-text search.

## Before you start

- No authentication. Server-side only (CORS is pinned to `https://sbgi.net`).
- One request per ten seconds. Responses cache for ten minutes.
- No rate-limit headers exist — there is no signal telling you to back off.

## Step 1 — Page the corpus

```
GET https://sbgi.net/wp-json/wp/v2/posts?per_page=100&page=1&orderby=date&order=desc
```

`listPosts` exposes 27 query parameters. The ones that matter:

| Parameter | Use |
|---|---|
| `per_page` | 1–100. **101 or more returns HTTP 400** `rest_invalid_param` with sub-code `rest_out_of_bounds`. |
| `page` | 1-based page index. |
| `search` | Full-text search across title and content. |
| `after` / `before` | ISO 8601 date bounds on publication date. |
| `modified_after` / `modified_before` | Date bounds on last modification — use these for change detection, not `after`. |
| `categories` / `categories_exclude` | Filter by category id, resolved via `listCategories`. |
| `orderby` / `order` | `date`, `modified`, `title`, `relevance`; `asc` or `desc`. |
| `_fields` | Comma-separated response trimming. Use it — the default payload is heavy. |

## Step 2 — Page correctly

Read the pagination signals from the **response headers**, not by guessing:

- `X-WP-Total` — total records matching the query (1753 unfiltered on 2026-08-12).
- `X-WP-TotalPages` — total pages at your `per_page`.
- `Link` — RFC 8288 header carrying `rel="next"` and `rel="prev"`. Follow it rather
  than incrementing `page` yourself; it stops cleanly at the end.

Requesting a page beyond `X-WP-TotalPages` returns an error, not an empty array.

## Step 3 — Trim the payload

The default post object is large: it inlines `content.rendered`, `yoast_head` (a full
HTML `<head>` block) and `yoast_head_json`. For monitoring you almost never need them.

```
GET https://sbgi.net/wp-json/wp/v2/posts?per_page=100&_fields=id,date,modified,slug,link,title,categories
```

## Step 4 — Detect change

Poll on `modified`, not `date` — Sinclair edits published releases:

```
GET https://sbgi.net/wp-json/wp/v2/posts?modified_after=2026-08-01T00:00:00&orderby=modified&order=desc&per_page=100
```

Store the high-water `modified` timestamp and use it as the next `modified_after`.
Because responses carry `cache-control: max-age=600` and there is no ETag or
Last-Modified on the JSON, conditional GET is unavailable — poll on the ten-minute
cache window, no faster.

## Step 5 — Search and retrieve

Cross-content search over posts and pages:

```
GET https://sbgi.net/wp-json/wp/v2/search?search=NextGen%20TV&per_page=20
```

`search` returns lightweight stubs (`id`, `title`, `url`, `type`, `subtype`). Resolve
a stub to the full record with `getPost`:

```
GET https://sbgi.net/wp-json/wp/v2/posts/7864
```

An unknown or non-post id returns HTTP 404 `rest_post_invalid_id`.

## Step 6 — Categories

```
GET https://sbgi.net/wp-json/wp/v2/categories?per_page=100
```

Resolve names to ids once and cache them; ids are bare integers with no type prefix and
are only meaningful within this collection. The `tags` collection exists but is empty on
this site — do not build a filter around it. `listComments` is likewise empty.

## Fields you cannot resolve

`author` is an integer id, but `/wp/v2/users` returns HTTP 401 `rest_user_cannot_view`.
Author identity is not obtainable. Do not attribute a release to a named person from
this API.

## Alternatives on the same host

- `getLatestPosts` (`/sbg/v1/latest-posts`) returns a **pre-rendered HTML fragment**,
  not structured data. Convenient for display, useless for analysis. Prefer `listPosts`.
- `https://sbgi.net/feed/` is a full RSS 2.0 feed — simpler, but no filtering, no date
  ranges and no total count.

## Error handling

Branch on `code`, never on `message`:

- `rest_invalid_param` (400) — read `data.params` for the per-parameter reason and
  `data.details[param].code` for the sub-code. Almost always `per_page` over 100 or a
  malformed date.
- `rest_post_invalid_id` (404) — resolve the id from `listPosts` first.
- `rest_no_route` (404) — wrong path.
