---
name: Track Elixir Medical clinical and corporate news
description: >-
  Pull Elixir Medical's press releases, clinical trial result summaries and corporate
  announcements from the company's own website content API, paginating correctly and
  filtering by category, without scraping HTML.
api: openapi/elixir-medical-wordpress-content-openapi.yml
operations:
  - getCategories
  - getPosts
  - getPostsId
  - getMedia
---

# Track Elixir Medical clinical and corporate news

Elixir Medical runs **no developer program**. There is no API key, no portal and no
support channel for this surface. What exists is the WordPress REST API of
`elixirmedical.com`, which serves the company's news and clinical-evidence posts as JSON.
Use it instead of scraping the rendered pages.

**Base URL:** `https://elixirmedical.com/wp-json`

## Authentication

None. Read operations on the content collections answer anonymously. Do not send
credentials — see `authentication/elixir-medical-authentication.yml`.

## Steps

1. **Resolve the category vocabulary first.** Call `getCategories` with `per_page=100`.
   Press releases, clinical evidence and careers postings all share the single `post`
   type on this install, so the category id is the only thing that separates them.
   Cache the id/slug pairs; do not hardcode ids.

2. **List posts.** Call `getPosts` with:
   - `categories` set to the ids you resolved in step 1,
   - `per_page=100` (the maximum this API accepts),
   - `orderby=date`, `order=desc`,
   - `_embed` if you want the author and featured image inlined in one round trip,
   - `after=<ISO 8601>` on repeat runs so you only fetch what is new.

3. **Paginate on the headers, not by guessing.** Read `X-WP-Total` and
   `X-WP-TotalPages` from the response, or follow the `Link: <...>; rel="next"` header.
   Stop when there is no `next` rel. Requesting a page beyond the last returns an error
   envelope, not an empty array.

4. **Fetch a single item** with `getPostsId` when you need the full rendered body.
   `content.rendered` and `excerpt.rendered` contain HTML — strip or sanitise before
   passing to a model.

5. **Resolve images** with `getMedia` (or read `_embedded['wp:featuredmedia']` if you
   used `_embed` in step 2). Use `source_url` from the media object.

## Rules

- **Pace your requests.** No rate limit is documented and no `RateLimit-*` or
  `Retry-After` header is returned, but the WP Engine edge in front of this site answers
  `403 text/html` under rapid sequential requests. Treat an HTML body on a JSON endpoint
  as back-pressure: wait and retry, do not parse it.
- **Respect the cache.** Anonymous reads carry `Cache-Control: max-age=600`. Polling
  faster than ten minutes gains you nothing.
- **Errors are not RFC 9457.** The envelope is `{code, message, data: {status}}` with
  `content-type: application/json`. Branch on `code`: `rest_no_route` means a bad path,
  `rest_post_invalid_id` a bad id, `rest_forbidden` a privileged route. See
  `errors/elixir-medical-problem-types.yml`.
- **Never write.** Write operations exist in the contract and require WordPress
  application-password credentials. This is a third-party company's website. Read only.
- **Language.** The site is published in Arabic, English, Indonesian and Vietnamese. Pass
  `wpml_language=en` to pin the language, or you will get mixed-locale results.
