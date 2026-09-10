---
name: browse-aee-fsw-catalog
description: Retrieve and filter the Aerospace Engineering Equipment (AEE) friction stir welding equipment catalogue — machines, automatic production lines and tooling — from the anonymous WordPress REST API at a-fsw.com.
api: aerospaceengineeringequipmentco:aee-products-api
operations:
  - listProductCategories
  - listProducts
  - getProduct
  - getMediaItem
---

# Browse the AEE friction stir welding catalogue

AEE publishes 18 products across 6 categories. There is no key, no signup and no rate-limit header — every
call below is an anonymous GET against `https://a-fsw.com/wp-json`.

## 1. Get the category map first

Run `listProductCategories` (`GET /wp/v2/product-category`). Ask for the fields you need:

```
GET https://a-fsw.com/wp-json/wp/v2/product-category?per_page=100&_fields=id,name,slug,count,link
```

At capture the terms were FSW Machine (id 3, 8 products), Pin Tools (id 13, 5), FSW Tools (id 2, 3) and
Automatic FSW Production Line (id 4, 2), plus two empty legacy terms whose `count` is 0. **Skip terms with
`count: 0`** — two of the six are dead, and their names duplicate live terms, so filtering on name rather
than id will silently return nothing.

## 2. List products, filtered by category

Run `listProducts` (`GET /wp/v2/product`). Filter with the taxonomy parameter, not a search string:

```
GET https://a-fsw.com/wp-json/wp/v2/product?product-category=3&per_page=100&_fields=id,slug,title,link,product-category
```

Pagination is `page` + `per_page`. `per_page` is capped at 100 — passing more returns HTTP 400
`rest_invalid_param` with `data.details.per_page.code = rest_out_of_bounds`. Read `X-WP-Total` and
`X-WP-TotalPages` from the response headers rather than counting pages yourself; both are CORS-exposed.

Free-text search is `?search=<term>` on the same collection.

## 3. Fetch one product

Run `getProduct` (`GET /wp/v2/product/{id}`) for the full record, including `content` (the rendered
specification prose) and `featured_media`.

```
GET https://a-fsw.com/wp-json/wp/v2/product/411
```

**Handle the 404 carefully.** A miss on this route does not return the WordPress JSON error envelope. The
edge intercepts it and returns an **nginx HTML 404 page** at `content-type: text/html`. Check the
content-type before parsing, or a missing id will surface as a JSON decode crash instead of a clean
not-found.

## 4. Get the image without a second round trip

`featured_media` is a media id, or `0` when no image is set. Rather than calling `getMediaItem` per
product, append `_embed` to the list call and read `_embedded['wp:featuredmedia']`:

```
GET https://a-fsw.com/wp-json/wp/v2/product?per_page=100&_embed
```

That collapses both the media and the taxonomy joins into one request.

## Conventions that apply throughout

- **Auth:** none. Do not send a credential; `?context=edit` returns HTTP 401 `rest_forbidden_context`.
- **Read-only:** every write method on these routes is refused anonymously. There is nothing to make
  idempotent and nothing to reverse.
- **No conditional GET:** no `ETag` or `Last-Modified` is served, so poll on `modified` / `modified_after`
  instead of relying on cache validators.
- **No published rate limit.** Nothing signals exhaustion. Be conservative anyway — the host is behind
  Cloudflare bot management, so an aggressive client should expect an interstitial rather than a 429.

Full conventions: `conventions/aerospaceengineeringequipmentco-conventions.yml`.
Error catalogue: `errors/aerospaceengineeringequipmentco-problem-types.yml`.
