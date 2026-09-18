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

A skill's slug comes from its own `Slug:` header field (see below); an agent's slug comes from its
filename (without `.md`). Importing is idempotent: re-importing the same slug updates the existing
local skill/agent instead of creating a duplicate, and a slug that already exists locally is shown
as "Installed" rather than importable again.

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

Four fixed sections, in this exact order, each introduced by a `--- Section ---` delimiter:

```markdown
--- Description ---

One paragraph describing what this agent is for.

--- Role ---

Who the agent is and how it should behave.

--- Goal ---

What a successful outcome looks like for this agent.

--- System Prompt ---

The full system prompt the agent runs with. Can be as long and as
structured (headings, lists, code blocks) as needed.
```

- **Title** is not part of the file — it's derived from the filename
  (`theme-design-expert.md` → "Theme Design Expert").
- All four sections are required; each becomes its matching agent field
  (description / role / goal / system prompt) as-is.

## Adding a new skill or agent

1. Drop a new `.md` file into `skills/` or `agents/`, following the format above.
2. For a skill, pick the `Slug:` value carefully — it's permanent as the item's identity once
   sites start importing it. For an agent, the filename plays that role instead, so pick it with
   the same care.
3. Commit and push. Sites check this repository live (with a short cache), so no release or
   versioning step is needed on this side.

## Updating an existing skill

Editing a skill file's content without bumping `Version:` has no effect on sites that already
imported it — they only pull an update when the Hub version is higher than what they have. So:

1. Edit the skill's body (and/or its `Description:`) as needed.
2. Bump `Version:` (e.g. `1.0.0` → `1.0.1` for a small fix, `1.1.0` for a bigger content change).
3. Commit and push.
