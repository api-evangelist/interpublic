---
name: interpublic-read-newsroom
description: >-
  Read the Interpublic Group corporate newsroom archive through the live
  WordPress REST API on the apex host - list and page through the 346 news
  posts, resolve their categories, tags and featured media, and pull a single
  post by id. Use when you need IPG corporate announcements as structured
  data rather than scraped HTML.
api: interpublic:wp-v2-api
base_url: https://interpublic.com
operations:
  - getWpV2Posts
  - getWpV2PostsById
  - getWpV2Categories
  - getWpV2Tags
  - getWpV2MediaById
  - getWpV2Search
generated: '2026-08-12'
method: generated
source: openapi/interpublic-wp-v2-api-openapi.yml
---

# Read the Interpublic Group newsroom

The Interpublic Group corporate site runs WordPress and still serves its REST
API from the **apex** host. Reads are anonymous — no credential exists or is
needed.

## Before you start

- **Use `https://interpublic.com`, never `https://www.interpublic.com`.** The
  www host 301-redirects every path to `https://www.omc.com/` following the
  Omnicom acquisition. So do the `Link: rel="next"` pagination headers this
  API returns — they are rewritten to the www host. **Rewrite the host back to
  the apex before following them**, or you will silently leave the API.
- The archive is **frozen**. The newest post is dated `2025-11-24`. Do not
  poll it for fresh news; treat it as an archive read.
- Errors come back as `{"code":…,"message":…,"data":{"status":…}}` with
  `application/json`. This is **not** RFC 9457. Match on `code`, never on
  `message`.

## Steps

1. **List posts.** `getWpV2Posts` — `GET /wp-json/wp/v2/posts`.
   Useful query parameters, all declared in the spec: `page` (default 1),
   `per_page` (default 10), `search`, `after`, `before`, `modified_after`,
   `categories`, `tags`, `orderby`, `order`, `_fields`, `_embed`.
   Read `X-WP-Total` (346 as of 2026-08-12) and `X-WP-TotalPages` from the
   response headers to size the walk before you make it.

2. **Keep the payload small.** Add `_fields=id,date,slug,link,title,excerpt`
   — the full record includes rendered `content`, `yoast_head` and `acf`
   blocks you almost never need.

3. **Resolve relationships in one call instead of many.** Add `_embed` and
   the response inlines author, terms and featured media under `_embedded`.
   Without it, follow the id-reference fields: `categories[]` →
   `getWpV2Categories`, `tags[]` → `getWpV2Tags`, `featured_media` →
   `getWpV2MediaById`. Note `author` cannot be resolved: `getWpV2Users`
   returns HTTP 401 `rest_user_cannot_view` on this install.

4. **Page through.** Increment `page` until `page > X-WP-TotalPages`. A page
   past the end returns HTTP 400 `rest_post_invalid_page_number`, so bound the
   loop on the header rather than on the error.

5. **Fetch one post.** `getWpV2PostsById` — `GET /wp-json/wp/v2/posts/{id}`.
   The canonical human URL is in `link`, but note those permalinks 301 to
   `investors.interpublic.com`, which answers HTTP 403 to automated clients.
   The API body is the reliable source of the text.

6. **Search across content types.** `getWpV2Search` — `GET
   /wp-json/wp/v2/search?search=…&type=post` returns lightweight
   `{id, title, url, type, subtype}` records, cheaper than filtering the
   posts collection when you only need to locate something.

## What you cannot do

- No writes. Every POST/PUT/DELETE requires a WordPress cookie session plus
  `X-WP-Nonce`, which third parties cannot obtain.
- No user directory, no settings, no revisions — all HTTP 401.
- No rate-limit signal is returned. Be conservative: the origin sits behind
  Cloudflare and WP Engine, and any throttle is undisclosed.
