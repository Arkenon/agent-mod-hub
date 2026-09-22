**Title:** WordPress Templates & Template Parts

**Slug:** wordpress-templates

**Description:** Use this skill for assembling clean, modular WordPress block templates and template parts through abilities, including overriding theme-provided templates by slug.

**Version:** 1.1.0

===

Block themes ship templates and template parts as read-only theme FILES. You can
never edit those files — but you do not need to: saving a record with the SAME
slug through the write abilities creates a database override that WordPress
automatically uses instead of the theme file. This is exactly how the Site
Editor works, and it is the correct way to customize any template, header or
footer. Content is serialized block markup — always apply the WordPress Block
Markup skill.

# Abilities

Templates (front-page, single, page, archive, …):
- `agent-mod/list-templates` — inventory with slugs; entries with a `post_id`
  already have a database override.
- `agent-mod/get-template` — full block markup by slug (`source` tells you if it
  comes from the theme file or a DB record).
- `agent-mod/add-or-update-template` — writes the DB record for a slug:
  updates the existing override, or creates one (also for hierarchy slugs the
  theme ships no file for, e.g. creating `front-page` from scratch).

Template parts (header, footer, sidebar, …):
- `agent-mod/list-template-parts` — inventory with slugs and areas.
- `agent-mod/get-template-part` — full block markup by slug.
- `agent-mod/add-or-update-template-part` — same override-by-slug behavior as
  templates. Editing the site header means updating the template PART with slug
  `header` — not a template.

# Workflow — follow these steps in order

1. **Read first.** The write abilities replace the whole document. Always fetch
   the current content with `agent-mod/get-template` /
   `agent-mod/get-template-part`, apply your edit to the full document, and send
   everything back. Keep the existing
   `<!-- wp:template-part {"slug":"header"} /-->` and footer references inside
   TEMPLATES intact — restyling the header/footer themselves is done by
   updating the template part, not by inlining markup into the template.
2. **Prefer updating over recreating.** Modify the existing content; never
   rebuild it from scratch when a small edit suffices. Never duplicate under a
   new slug when the task is to change the existing header/footer/template —
   write to the SAME slug so the override takes effect.
3. **Compose from patterns.** Never build page layouts directly inside
   templates — templates should mainly reference reusable patterns and template
   parts. If the layout you need doesn't exist as a pattern yet, create the
   pattern first (see the Patterns skill), then reference it using the correct
   syntax for its source (see "Referencing patterns" below).
4. **Validate, write, verify.** Run `agent-mod/validate-block-markup` on the
   full markup, fix every issue, write with the add-or-update ability, then
   re-read to confirm (`source` should now be the DB record / `post_id` set).
   The validator does NOT check that pattern slugs or block refs resolve —
   verify those yourself against `agent-mod/list-patterns` output.

# Referencing patterns — the #1 source of empty templates

`<!-- wp:pattern {"slug":"…"} /-->` resolves ONLY against the block pattern
REGISTRY (patterns registered in PHP by the theme/plugin — the
`registry_patterns` array of `agent-mod/list-patterns`). Registry slugs are
always namespaced, e.g. `twentytwentyfive/hero`.

Patterns created through `agent-mod/create-pattern` / `duplicate-pattern` are
DATABASE patterns (`wp_block` posts, the `database_patterns` array). They have
a `post_id` and NO registry slug. Referencing them with `wp:pattern` — or with
an invented slug like `my-hero` — silently fails: the editor shows a
"Pattern Placeholder" in List View and the front end renders nothing.

Use the syntax that matches the pattern's source:

| Source (from `list-patterns`) | Reference in a template |
| --- | --- |
| `registry_patterns` (has `name`/slug) | `<!-- wp:pattern {"slug":"theme/pattern-name"} /-->` |
| `database_patterns`, `sync_status: synced` | `<!-- wp:block {"ref":123} /-->` (123 = `post_id`) |
| `database_patterns`, `sync_status: unsynced` | Not referenceable — inline the pattern's markup |

Rules that follow from this:

- NEVER invent a pattern slug. Only use a slug that appears verbatim in
  `registry_patterns`.
- ALWAYS create patterns as `sync_status: "synced"` unless the user explicitly
  asked otherwise, and reference them by their returned `post_id` via
  `wp:block`. Synced is the only status a template can reference, and it keeps
  one source of truth — editing the pattern updates every template using it.
  `create-pattern` defaults to `unsynced`, so pass the parameter explicitly.
- If an existing database pattern you need is `unsynced`, prefer converting the
  workflow to a synced pattern over inlining a copy into the template, unless
  the user wants a one-off variation.
- Unsynced database patterns are insert-only: fetch them with
  `agent-mod/get-pattern` (`source: "database"`, `post_id`) and paste the
  returned markup into the template body. Cross-check every `post_id` you write
  against `list-patterns` before saving.
- After writing a template that references patterns, re-read it and confirm
  each reference is a real registry slug or a real `wp_block` `post_id` —
  a template that saved successfully can still render blank.

# Template anatomy

- Use template hierarchy slugs: `index`, `home`, `front-page`, `single`,
  `page`, `archive`, `search`, `404` (plus specific ones like
  `single-{post_type}` or `page-{slug}`). List existing templates first and
  prefer updating an existing record; creating a hierarchy slug the theme has
  no file for is allowed and supported.
- Content templates (single, page) must contain `<!-- wp:post-content /-->`
  (usually inside a constrained group) — without it the page body never renders.
- Archive-style templates use a Query Loop that inherits the main query:
  `{"query":{...,"inherit":true}}`.
- Maintain consistent spacing and layout hierarchy: one top-level structure of
  header template-part → main content group → footer template-part.

# Rules

- Keep templates clean and modular; reuse existing patterns whenever possible.
- Use theme preset slugs for any styling; call `agent-mod/get-global-styles`
  (read-only) before styling anything. Do NOT call
  `agent-mod/update-global-styles` as part of template work — global styles are
  only changed when the user explicitly asks for a site-wide style change.
