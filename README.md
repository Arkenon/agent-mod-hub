# agent-mod-hub

Shared content for the AgentMod Pro WordPress plugin, published here so it can be browsed and imported
straight into a site's Hub (e.g. **Skill Hub**, **Agent Hub**) — no manual export/import round-trip
required.

Any site running AgentMod Pro can point its Hub at this repository (or a fork of it) to list what's
available and import what it doesn't have yet, matched by slug.

## Scope

This repository currently covers two item types:

```
skills/*.md   -> imported as a Skill  (agentmod_skill)
agents/*.md   -> imported as an Agent (agentmod_agent)
```

More types are expected to be added over time (e.g. widgets, scheduled tasks) as AgentMod grows —
each would get its own top-level folder alongside `skills/` and `agents/`, following the same
overall pattern described below. See each folder's own README for its file format:

- [`skills/README.md`](skills/README.md) — how to write a Skill `.md` file.
- [`agents/README.md`](agents/README.md) — how to write an Agent `.md` file.

## Shared conventions

These apply to every item type in this repository, regardless of folder:

- Every file has a `Slug:` header field, which is the item's **permanent identity**. Pick it
  carefully — once a site imports a slug, that's what ties the local copy back to the Hub entry.
- Every file has a `Version:` field, a semantic version (`MAJOR.MINOR.PATCH`, starting at `1.0.0`),
  compared with PHP's `version_compare()`.
- Importing is idempotent and version-gated: re-importing the same slug updates the existing local
  item instead of creating a duplicate, and a slug already present locally shows as "Installed" (or
  "Update" once the Hub copy's version is higher) rather than importable again. Editing a file's
  content without bumping `Version:` has no effect on sites that already imported it.
- A header block (`Field: value` lines) is terminated by a line containing only `===`, followed by
  the item's free-form body/content, kept exactly as written.

## Adding or updating an item

1. Drop a new `.md` file into the relevant type folder (e.g. `skills/`, `agents/`), following that
   folder's README.
2. Pick `Slug:` carefully for a new item — it's permanent once sites start importing it.
3. For an edit to an existing item, bump `Version:` (e.g. `1.0.0` → `1.0.1` for a small fix, `1.1.0`
   for a bigger content change) — otherwise sites that already imported it won't see the update.
4. Commit and push. Sites check this repository live (with a short cache), so no release or
   versioning step is needed on this side.
