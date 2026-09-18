# agent-mod-hub

Skills and agents for the AgentMod WordPress plugin, published here so they can be browsed and
imported straight into a site's **Skill Hub / Agent Hub** — no manual export/import round-trip
required.

Any site running AgentMod Pro can point its Hub at this repository (or a fork of it) to list what's
available and import what it doesn't have yet, matched by slug.

## Structure

```
skills/*.md   -> imported as a Skill  (agentmod_skill)
agents/*.md   -> imported as an Agent (agentmod_agent)
```

Both a skill's and an agent's slug come from their own `Slug:` header field (see below).
Importing is idempotent: re-importing the same slug updates the existing local skill/agent instead
of creating a duplicate, and a slug that already exists locally is shown as "Installed" (or
"Update" once the Hub copy's version is higher) rather than importable again.

## File formats

### Skills (`skills/*.md`)

A `Title:` / `Slug:` / `Description:` / `Version:` header, terminated by a line containing only
`===`, followed by the skill's body:

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
  body — so a heading inside the content (e.g. `# Mental model`) is never mistaken for the
  skill's title.
- **Slug** is the item's permanent identity (see "Structure" above) — set it once, keep it stable.
- **Version** is a semantic version, `MAJOR.MINOR.PATCH` (starts at `1.0.0`), compared with PHP's
  `version_compare()`. Bump it whenever you change a skill's content — a site only overwrites its
  locally installed copy when the Hub version is higher than what it already has; otherwise the
  skill stays listed as up to date and is not re-imported.
- **Content** is everything after the `===` separator line, unchanged.

### Agents (`agents/*.md`)

An explicit header of `Field: value` lines, terminated by a line containing only `===`, followed by
the agent's full System Prompt:

```markdown
Title: Agent Title
Slug: agent-slug
Icon: art
Description: One paragraph shown in the agent tray.
Role: Who the agent is and how it should behave.
Goal: What a successful outcome looks like for this agent.
Version: 1.0.0
Model: use_default
Ability Source: selected
Abilities: core/get-site-info, agent-mod/get-global-styles, agent-mod/validate-block-markup
Skills: wordpress-templates, wordpress-patterns
===

The full system prompt the agent runs with. Can be as long and as
structured (headings, lists, code blocks) as needed.
```

- **Title**, **Slug**, **Icon**, **Description**, **Role**, **Goal**, and **Version** map directly
  to the matching agent field. Fields can appear in any order — each one's value stops at the next
  known field label, not at the end of the header.
- **Icon** is a [Dashicon slug](https://developer.wordpress.org/resource/dashicons/) (e.g. `art`),
  shown as the agent's avatar. Leave empty to use the default agent image.
- **Model** is either `use_default` (inherit the site's default model) or a specific
  `provider:model` value the site already supports.
- **Ability Source** is `all` (every registered ability) or `selected` (only the `Abilities:` list).
- **Abilities** is a comma-separated list of ability names exactly as registered (e.g.
  `agent-mod/get-global-styles`), used only when `Ability Source: selected`.
- **Skills** is a comma-separated list of skill **slugs** — not local post IDs, since a Hub file
  cannot know another site's IDs. Each slug is resolved against the importing site's own skills;
  a slug the site does not have is silently skipped rather than failing the whole import.
- **System Prompt** is everything after the `===` separator line, unchanged.

## Adding a new skill or agent

1. Drop a new `.md` file into `skills/` or `agents/`, following the format above.
2. Pick the `Slug:` value carefully in either case — it's permanent as the item's identity once
   sites start importing it.
3. Commit and push. Sites check this repository live (with a short cache), so no release or
   versioning step is needed on this side.

## Updating an existing skill or agent

Editing a file's content without bumping `Version:` has no effect on sites that already imported
it — they only pull an update when the Hub version is higher than what they have. So:

1. Edit the body (and/or the header fields) as needed.
2. Bump `Version:` (e.g. `1.0.0` → `1.0.1` for a small fix, `1.1.0` for a bigger content change).
3. Commit and push.
