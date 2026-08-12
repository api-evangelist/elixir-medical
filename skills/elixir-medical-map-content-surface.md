---
name: Map the Elixir Medical content surface before querying it
description: >-
  Discover what this WordPress-backed API actually exposes — post types, taxonomies,
  statuses and the treating-physician locator — from the site's own self-describing
  registries, rather than assuming the WordPress defaults.
api: openapi/elixir-medical-wordpress-content-openapi.yml
operations:
  - getTypes
  - getTaxonomies
  - getWpfmDesignations
  - getWpfmLocations
  - getSearch
---

# Map the Elixir Medical content surface before querying it

This API is self-describing. Before writing any query, read the registries — the install
carries twenty post types and five taxonomies, most of which are page-builder plumbing
rather than content. Guessing the shape wastes calls and produces wrong results.

**Base URL:** `https://elixirmedical.com/wp-json` — no authentication for reads.

## Steps

1. **List the post types.** Call `getTypes`. The response is an object keyed by type
   slug; each value carries `name`, `rest_base`, `hierarchical` and `taxonomies`.
   Build your paths from `rest_base` — it is not always the same as the slug.

2. **Discard the plumbing.** Ignore `wp_block`, `wp_template`, `wp_template_part`,
   `wp_global_styles`, `wp_navigation`, `wp_font_family`, `wp_font_face`,
   `nav_menu_item`, `elementor_library`, `elementor_snippet`, `elementskit_content`,
   `elementskit_template`, `popup`, `popup_theme` and `pum_cta`. They are editor and
   page-builder artifacts. The content types are `post`, `page`, `attachment` and
   `wpfm_locations`.

3. **List the taxonomies.** Call `getTaxonomies` and read the `types` array on each one
   to learn which post type it filters. On this install: `category` and `post_tag` apply
   to `post`; `wpfm_designations` applies to `wpfm_locations`.

4. **Walk the locator.** `wpfm_locations` is the one genuinely domain-specific entity —
   the custom post type behind the site's "Find a Doctor" feature. Call
   `getWpfmDesignations` first to resolve the designation vocabulary, then
   `getWpfmLocations` filtered by it.

5. **Use search for cross-type lookups.** `getSearch` projects across post types and
   terms in one call and returns a light `{id, title, url, type, subtype}` shape. Use it
   to locate an object, then fetch the full record from its own collection.

## Rules

- **Read the registries, do not assume WordPress defaults.** Custom post types and their
  `rest_base` values are install-specific.
- **`_fields` keeps responses small.** Ask for `_fields=id,slug,title,link,date` when you
  are building an index; the full objects are large.
- **`context=view` is the only context available anonymously.** Requesting
  `context=edit` returns `rest_forbidden` (401).
- **This is a third-party corporate website.** Read only, pace requests, and treat an
  HTML body on a JSON endpoint as edge back-pressure rather than data.
