# Skills (`skills/*.md`)

A Skill file has a `Title:` / `Slug:` / `Description:` / `Version:` header, terminated by a line
containing only `===`, followed by the skill's body:

```markdown
Title: Skill Title
Slug: skill-slug
Description: One-paragraph description shown in the skill tray.
Version: 1.0.0
===

Free-form Markdown body — the text an agent receives as skill content when it
loads this skill. Code blocks, block markup, and other characters are kept
exactly as written (nothing here is escaped or re-encoded).
```

- **Title**, **Slug**, **Description**, and **Version** are explicit fields, not derived from the
  body — so a heading inside the content (e.g. `# Mental model`) is never mistaken for the skill's
  title.
- **Slug** is the item's permanent identity — set it once, keep it stable. A site matches Hub
  entries to its own local skills by this value.
- **Version** is a semantic version, `MAJOR.MINOR.PATCH` (starts at `1.0.0`), compared with PHP's
  `version_compare()`. Bump it whenever you change a skill's content — a site only overwrites its
  locally installed copy when the Hub version is higher than what it already has; otherwise the
  skill stays listed as up to date and is not re-imported.
- **Content** is everything after the `===` separator line, unchanged.

## Adding a new skill

1. Drop a new `.md` file into `skills/`, following the format above.
2. Pick the `Slug:` value carefully — it's permanent as the item's identity once sites start
   importing it.
3. Commit and push. Sites check this repository live (with a short cache), so no release or
   versioning step is needed on this side.

## Updating an existing skill

Editing a file's content without bumping `Version:` has no effect on sites that already imported
it — they only pull an update when the Hub version is higher than what they have. So:

1. Edit the body (and/or the header fields) as needed.
2. Bump `Version:` (e.g. `1.0.0` → `1.0.1` for a small fix, `1.1.0` for a bigger content change).
3. Commit and push.
