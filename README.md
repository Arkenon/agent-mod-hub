# agent-mod-hub

Skills and agents for the AgentMod WordPress plugin, published here so they can be browsed and
imported straight into a site's **Skill Hub / Agent Hub** — no manual export/import round-trip
required.

Any site running AgentMod Pro can point its Hub at this repository (or a fork of it) to list what's
available and import what it doesn't have yet, matched by filename.

## Structure

```
skills/*.md   -> imported as a Skill  (agentmod_skill)
agents/*.md   -> imported as an Agent (agentmod_agent)
```

The filename (without `.md`) becomes the item's **slug**. Importing is idempotent: re-importing the
same slug updates the existing local skill/agent instead of creating a duplicate, and a slug that
already exists locally is shown as "Installed" rather than importable again.

## File formats

### Skills (`skills/*.md`)

A single `#` heading followed by the skill's body:

```markdown
# Skill Title

Free-form Markdown body — the text an agent receives as skill content when it
loads this skill. Code blocks, block markup, and other characters are kept
exactly as written (nothing here is escaped or re-encoded).
```

- **Title** comes from the `# Heading`.
- **Content** is everything after the heading, unchanged.
- **Description** is not a separate field — the importer derives it automatically from the first
  paragraph of the body (trimmed to ~150 characters), so no extra metadata is required.

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
2. Pick the filename carefully — it's permanent as the item's slug once sites start importing it.
3. Commit and push. Sites check this repository live (with a short cache), so no release or
   versioning step is needed on this side.
