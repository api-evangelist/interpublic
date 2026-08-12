---
name: interpublic-discover-surface
description: >-
  Discover exactly what is callable on Interpublic Group's own hosts before
  assuming anything - read the WordPress route-discovery document that
  enumerates all 254 routes across 17 namespaces, separate the anonymous
  routes from the capability-gated ones, and recognise the two dead ends
  (the www redirect to omc.com and the unrouted Apigee gateway).
api: interpublic:root-api
base_url: https://interpublic.com
operations:
  - getWpJsonIndex
  - getWpV2Types
  - getWpAbilitiesV1Abilities
  - getOembed10Embed
generated: '2026-08-12'
method: generated
source: openapi/interpublic-root-api-openapi.yml
---

# Discover the Interpublic Group API surface

Interpublic Group publishes no developer portal, no product API and no
OpenAPI. What it does serve is a self-describing WordPress REST API on the
apex host. Start there — that document, not documentation, is authoritative.

## Steps

1. **Read the index.** `getWpJsonIndex` — `GET https://interpublic.com/wp-json/`.
   HTTP 200, ~258 KB. It returns:
   - `name` / `description` — both `"IPG"`; `url` and `home` —
     `https://www.interpublic.com`. This is the ownership proof that the
     surface belongs to Interpublic Group and not to a sibling brand.
   - `namespaces[]` — 17 of them.
   - `routes{}` — 254 entries, each with its `methods` and full `args`
     (name, type, required, default, enum, description). This is the machine-
     readable parameter contract; the four specs in `openapi/` are a
     mechanical conversion of it.
   - `authentication` — an empty array, meaning no authentication plugin is
     registered. There is no credential to request.

2. **Split core from plugin noise.** Only four namespaces are worth calling:
   `wp/v2` (114 routes), `wp-abilities/v1` (6), `oembed/1.0` (3) and the root
   (2). The other thirteen — `yoast/v1`, `redirection/v1`, `webdev-insight/v1`,
   `wordfence/v1`, `accessibility-suite/v1`, `aam/v2`, `wp-offload-media/v1`,
   `wpe_sign_on_plugin/v1`, `wpe/cache-plugin/v1`, `wr/v1`, `fluent-smtp`,
   `ada-plugin/v1`, `wp-site-health/v1`, `wp-block-editor/v1` — are vendor
   plugin internals, not a published API.

3. **Learn the content types.** `getWpV2Types` — `GET /wp-json/wp/v2/types`.
   Alongside WordPress core types this install registers three custom ones:
   `ci_lead_form`, `supplier_lead_form` and `icons`.

4. **Probe the gate, do not guess it.** `getWpAbilitiesV1Abilities` —
   `GET /wp-json/wp-abilities/v1/abilities` returns HTTP 401
   `rest_forbidden`. A nonsense path on the same host returns HTTP 404
   `rest_no_route`, so the host **discriminates**: the Abilities surface
   genuinely exists and is capability-gated, rather than merely absent. Apply
   the same control-path test before recording anything else as "gated".

5. **Know the dead ends.**
   - `https://www.interpublic.com/*` — every path 301s and terminates at
     `https://www.omc.com/` with HTML. A 200 there is the Omnicom homepage,
     not the document you asked for. Never accept it as a hit.
   - `https://api.interpublic.com/*` — a live Apigee gateway that answers
     every path, including a nonsense control, with
     `messaging.adaptors.http.flow.ApplicationNotFound`. A gateway is deployed
     but no proxy is publicly bound to it.
   - `https://investors.interpublic.com/*` — HTTP 403 to everything including
     a control. Bot-blocked; infer nothing from it.
   - `/.well-known/*` and `/llms.txt` — 404 on the apex, 301-to-omc.com on
     www. Nothing is served.

6. **Embed a page, if you need a card.** `getOembed10Embed` — `GET
   /wp-json/oembed/1.0/embed?url=…` returns an oEmbed representation of any
   URL on the site. HTTP 200 anonymously.
