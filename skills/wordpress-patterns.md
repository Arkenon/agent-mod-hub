**Title:** WordPress Block Patterns

**Slug:** wordpress-patterns

**Description:** Use this skill for creating, duplicating, and updating reusable, responsive WordPress Block Patterns through abilities. Patterns are serialized block markup stored in the database never PHP pattern files.

**Version:** 1.1.0

===

Patterns on this site live in the database and are managed exclusively through
abilities (`agent-mod/list-patterns`, `agent-mod/get-pattern`,
`agent-mod/create-pattern`, `agent-mod/update-pattern`,
`agent-mod/duplicate-pattern`). There are no PHP file headers, no
`register_block_pattern()` calls, and no i18n/escaping functions the content is
pure serialized block markup. Always apply the WordPress Block Markup skill when
writing it.

# Workflow follow these steps in order

1. **Triage.** List existing patterns with `agent-mod/list-patterns` and inspect
   candidates with `agent-mod/get-pattern`.
   - If a suitable pattern exists, duplicate it with `agent-mod/duplicate-pattern`
     and update the copy never rebuild from scratch what already exists.
   - Only create a new pattern when nothing suitable exists.
2. **Design decisions.** Before writing markup, call `agent-mod/get-global-styles`
   and decide deliberately:
   - Purpose: what section is this (hero, features, CTA, testimonial, footer)?
   - Tone: calm/dense, light/dark expressed only through the theme's preset
     slugs, never hard-coded colors when a preset fits.
   - Spacing rhythm: pick a consistent spacing preset scale for section padding
     and blockGap (tight / standard / generous) and stick to it.
   - Hierarchy: one clear heading level and size progression; asymmetric column
     splits (66/33) usually beat equal ones.
   - Width: read `settings.layout.contentSize` / `wideSize` from global styles.
     Outer section groups get `"align":"full"`; inner card grids, image rows and
     multi-column content that the reference design shows wider than the text
     column get `"align":"wide"` (class `alignwide`). Reserve the default content
     width for prose. If a design screenshot shows content clearly wider than a
     text column, that is the wide width use it.
3. **Plan the block tree.** Outline the nesting (group, columns ) before
   writing any markup. Top level is usually a single `group` with
   `{"layout":{"type":"constrained"}}` and often `{"align":"full"}`.
4. **Write.** Produce the full serialized markup, validate it with
   `agent-mod/validate-block-markup`, fix every issue, then call
   `agent-mod/create-pattern` (or `update-pattern`). ALWAYS pass
   `sync_status: "synced"` unless the user explicitly asked for an unsynced
   pattern — the ability's own default is `unsynced`, so you must set this
   every time (see "Always create synced patterns" below).
5. **Verify.** Re-read with `agent-mod/get-pattern` and confirm the stored markup
   matches what you sent before reporting success. Record the returned
   `post_id` — that, not a slug, is how the pattern is referenced elsewhere.

# Patterns created here have NO slug

`create-pattern` and `duplicate-pattern` write `wp_block` posts. They are
identified by `post_id` only. They are NOT added to the block pattern registry,
so `<!-- wp:pattern {"slug":"…"} /-->` can never point at them — that block
resolves slugs against registry (theme/plugin PHP) patterns exclusively, and an
unresolved slug renders as a "Pattern Placeholder" in the editor and as nothing
on the front end.

To use a pattern you created:

- `sync_status: "synced"` → reference it with `<!-- wp:block {"ref":POST_ID} /-->`.
- `sync_status: "unsynced"` → it is insert-only: read it back with
  `agent-mod/get-pattern` and paste its markup inline where you need it.

Never report a pattern as "added to a template" without having written either a
real `post_id` ref or the inlined markup.

# Always create synced patterns

Default to `sync_status: "synced"` for every pattern you create or duplicate.
Only use `unsynced` when the user explicitly asks for it (or asks for a
one-off/editable-per-instance copy).

Why synced is the better default here:

- It is the only sync status that can be referenced from templates and
  template parts (`<!-- wp:block {"ref":POST_ID} /-->`), which is exactly how
  templates are supposed to be composed.
- One source of truth: editing the pattern updates every place it is used, so
  a site-wide section change is a single `update-pattern` call instead of an
  edit per template.
- Referenced-by-id means a template stays small and readable, and re-reading it
  shows intent rather than a wall of duplicated markup.

Because `create-pattern` defaults to `unsynced`, omitting the parameter is a
bug, not a shortcut — pass `sync_status: "synced"` explicitly and confirm the
returned `sync_status` in the response.

# Rules

- Build reusable patterns: avoid page-specific content (real product names,
  dates, one-off copy) inside patterns; use generic placeholder copy instead.
- Keep layouts responsive: constrained/flex/grid layouts and preset spacing 
  never fixed pixel widths on content unless the design requires it.
- Use theme design tokens (preset slugs) whenever possible; fall back to raw
  values only when no preset fits.
- Give patterns clear, descriptive titles that state what they are
  ("Hero image right, dark", "Pricing 3 columns").
- IMAGES IN PATTERNS: if no image URL or attachment ID is provided, do not invent
  one and do not hotlink external placeholders. Leave the Image block empty.
