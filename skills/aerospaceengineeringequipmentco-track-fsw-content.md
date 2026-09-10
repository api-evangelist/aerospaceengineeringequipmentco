---
name: track-aee-fsw-content
description: Monitor and retrieve Aerospace Engineering Equipment (AEE) news announcements and friction stir welding technical articles, and search across the whole a-fsw.com site, using the anonymous WordPress REST API.
api: aerospaceengineeringequipmentco:aee-content-api
operations:
  - listCategories
  - listPosts
  - getPost
  - listPages
  - search
---

# Track AEE news and FSW technical content

AEE runs an actively maintained editorial surface: 50 posts at capture, split into News (4) and Blog (46),
plus 14 static pages. All of it is anonymously readable at `https://a-fsw.com/wp-json`.

## 1. Resolve the two categories

Run `listCategories` (`GET /wp/v2/categories`):

```
GET https://a-fsw.com/wp-json/wp/v2/categories?_fields=id,name,slug,count
```

At capture: Blog is id 14 (46 posts), News is id 1 (4 posts). Resolve by `slug`, not by name — names are
display strings and are not guaranteed unique on this site, as the duplicated product-category names show.

## 2. Poll for new posts

Run `listPosts` (`GET /wp/v2/posts`) with a date filter rather than fetching everything each time. The
collection is ordered newest-first by default.

```
GET https://a-fsw.com/wp-json/wp/v2/posts?after=2026-09-01T00:00:00&per_page=100&_fields=id,date,modified,slug,title,link,categories
```

- `after` / `before` filter on publication date; `modified_after` / `modified_before` filter on last edit.
  **Use `modified_after` to catch silent revisions** — AEE edits published posts, and a publication-date
  poll will miss those.
- Restrict to announcements with `&categories=1`, or to technical articles with `&categories=14`.
- There is no `ETag` or `Last-Modified` on this API, so a stored high-water mark on `modified` is the only
  workable incremental strategy.

## 3. Fetch the article body

Run `getPost` (`GET /wp/v2/posts/{id}`). `content.rendered` is HTML; `excerpt.rendered` is the summary.
As with products, an id miss returns an **nginx HTML 404**, not JSON — branch on content-type.

## 4. Read the static company and process pages

Run `listPages` (`GET /wp/v2/pages?per_page=100`). These carry the durable material: the company profile,
the FSW and FSSW process explainers, the industry pages (aerospace and aviation, railway, automotive,
marine engineering, shipbuilding, power electronics), service, FAQ, contact and privacy policy. They change
rarely, so fetch them once and refresh on `modified`.

**Do not guess page URLs.** `https://a-fsw.com/products/` and `https://a-fsw.com/company-profile/` both
return HTTP 200 serving the homepage — the site soft-404s every unknown path. The pages collection is the
only reliable index of what actually exists.

## 5. Search across everything at once

Run `search` (`GET /wp/v2/search`) when you do not know which collection holds the answer:

```
GET https://a-fsw.com/wp-json/wp/v2/search?search=friction%20stir&per_page=20
```

Hits are lightweight — `id`, `title`, `url`, `type`, `subtype`. Use `subtype` to decide which collection to
re-fetch the full record from (`post`, `page` or `product`). Ids are not namespaced across types, so the
`subtype` is required to resolve a hit, not optional.

## Conventions that apply throughout

- **Auth:** none, and none is available. `?context=edit` returns HTTP 401.
- **Pagination:** `page` + `per_page` (max 100); read `X-WP-Total`, `X-WP-TotalPages` and the RFC 8288
  `Link` header.
- **No rate-limit signal** is published or observable; throttle yourself.
- AEE publishes no changelog, no status page and no deprecation policy. This surface carries no
  availability commitment, and a WordPress core upgrade can reshape `wp/v2` without notice.

Full conventions: `conventions/aerospaceengineeringequipmentco-conventions.yml`.
Data model: `data-model/aerospaceengineeringequipmentco-data-model.yml`.
