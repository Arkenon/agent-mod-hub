# Agents (`agents/*.md`)

An Agent file has an explicit header of `Field: value` lines, terminated by a line containing only
`===`, followed by the agent's full System Prompt:

```markdown
**Title:** Agent Title

**Slug:** agent-slug

**Icon:** art

**Description:** One paragraph shown in the agent tray.

**Role:** Who the agent is and how it should behave.

**Goal:** What a successful outcome looks like for this agent.

**Version:** 1.0.0

**Model:** use_default

**Ability Source:** selected

**Abilities:** core/get-site-info, agent-mod/get-global-styles, agent-mod/validate-block-markup

**Skills:** wordpress-templates, wordpress-patterns

**Agent Types:** theme-design

**Personalities:** meticulous, technical

===

The full system prompt the agent runs with. Can be as long and as
structured (headings, lists, code blocks) as needed.
```

Wrapping each label in `**` (bold) and separating fields with a blank line is purely
cosmetic — it renders the header as legible bold key/value pairs instead of one
run-together paragraph. The parser accepts labels with or without the `**`
wrapper and is indifferent to blank lines, so plain `Title: Agent Title` on
consecutive lines (no bold, no blank lines) still parses identically. Pick
whichever you find easier to read; just keep each field's `Label:` at the
start of its own line.

- **Title**, **Slug**, **Icon**, **Description**, **Role**, **Goal**, and **Version** map directly
  to the matching agent field. Fields can appear in any order — each one's value stops at the next
  known field label, not at the end of the header.
- **Slug** is the item's permanent identity — set it once, keep it stable. A site matches Hub
  entries to its own local agents by this value.
- **Icon** is a [Dashicon slug](https://developer.wordpress.org/resource/dashicons/) (e.g. `art`),
  shown as the agent's avatar. Leave empty to use the default agent image.
- **Version** is a semantic version, `MAJOR.MINOR.PATCH` (starts at `1.0.0`), compared with PHP's
  `version_compare()`. Bump it whenever you change the agent — a site only overwrites its locally
  installed copy when the Hub version is higher than what it already has.
- **Model** is either `use_default` (inherit the site's default model) or a specific
  `provider:model` value the site already supports.
- **Ability Source** is `all` (every registered ability) or `selected` (only the `Abilities:` list).
- **Abilities** is a comma-separated list of ability names exactly as registered (e.g.
  `agent-mod/get-global-styles`), used only when `Ability Source: selected`.
- **Skills** is a comma-separated list of skill **slugs** — not local post IDs, since a Hub file
  cannot know another site's IDs. Each slug is resolved against the importing site's own skills; a
  slug the site does not have is silently skipped rather than failing the whole import.
- **Agent Types** and **Personalities** are comma-separated names or slugs for the matching
  taxonomy. Unlike Skills, an unmatched value here is not skipped — it's **created** as a new term
  on the importing site, since a fresh site may simply not have that type/trait yet.
- **System Prompt** is everything after the `===` separator line, unchanged.

## Adding a new agent

1. Drop a new `.md` file into `agents/`, following the format above.
2. Pick the `Slug:` value carefully — it's permanent as the item's identity once sites start
   importing it.
3. Commit and push. Sites check this repository live (with a short cache), so no release or
   versioning step is needed on this side.

## Updating an existing agent

Editing a file's content without bumping `Version:` has no effect on sites that already imported
it — they only pull an update when the Hub version is higher than what they have. So:

1. Edit the header fields and/or system prompt as needed.
2. Bump `Version:` (e.g. `1.0.0` → `1.0.1` for a small fix, `1.1.0` for a bigger content change).
3. Commit and push.
