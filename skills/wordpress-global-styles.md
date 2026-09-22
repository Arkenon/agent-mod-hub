**Title:** WordPress Global Styles

**Slug:** wordpress-global-styles

**Description:** Use this skill to update global styles (theme.json or global style data saved in the database) safely without overwriting existing structure or presets.

**Version:** 1.1.0

===

--------------------------------------------------
# READ vs WRITE — when to use which tool
--------------------------------------------------
`agent-mod/get-global-styles` is for READING the design system (preset slugs,
palette, typography) before styling patterns, templates or template parts — call
it freely. `agent-mod/update-global-styles` CHANGES the whole site's styles:
only call it when the user explicitly asks to change global/site-wide styles
(palette, fonts, gradients). Never call it as a side step of creating or
editing a pattern, template, or template part — in those tasks you only read
presets and reference their slugs in block markup.

--------------------------------------------------
# GENERAL PRINCIPLES
--------------------------------------------------
Always call `agent-mod/get-global-styles` and analyze the current Global Styles before making changes — and before writing ANY color, font-size, or spacing value into block markup: use only the preset slugs the theme actually defines. Understand the existing design system before modifying it. Preserve the structure of the active theme. Never recreate the theme's design system from scratch. Never overwrite unrelated settings. Apply the smallest possible valid changes.

--------------------------------------------------
# WRITE CONTRACT
--------------------------------------------------
Writes go through `agent-mod/update-global-styles`. Send only the keys you are changing; everything else is preserved for you. The payload shape is exactly what `agent-mod/get-global-styles` returns, so read first and mirror that shape back.

Presets are FLAT LISTS of objects. Each entry needs a non-empty `slug` plus its value key:

- `settings.color.palette` -> `{"slug":"primary","name":"Primary","color":"#3d35e8"}`
- `settings.color.gradients` -> `{"slug":"hero","name":"Hero","gradient":"linear-gradient(...)"}`
- `settings.color.duotone` -> `{"slug":"duo","name":"Duo","colors":["#000","#fff"]}`
- `settings.typography.fontSizes` -> `{"slug":"large","name":"Large","size":"1.5rem"}`
- `settings.typography.fontFamilies` -> `{"slug":"body","name":"DM Sans","fontFamily":"\"DM Sans\", sans-serif"}`
- `settings.spacing.spacingSizes` -> `{"slug":"50","name":"Medium","size":"1.5rem"}`
- `settings.shadow.presets` -> `{"slug":"soft","name":"Soft","shadow":"0 1px 2px rgba(0,0,0,.08)"}`

Hard rules:

1. Never key a preset list by origin. `default`, `theme` and `custom` are WordPress internals; a payload like `{"palette":{"theme":[...]}}` is wrong, and writing a `theme` or `default` origin would overwrite the active theme's own presets.
2. Never put a number, a boolean or a string where a preset list belongs. `"duotone": 0` or `"spacingSizes": 0` corrupts Global Styles and produces PHP warnings on every page of the site.
3. The on/off switches are SIBLINGS of the preset lists, never the lists themselves: `settings.color.custom`, `settings.color.defaultPalette`, `settings.color.defaultGradients`, `settings.color.defaultDuotone`, `settings.typography.defaultFontSizes`, `settings.spacing.defaultSpacingSizes`. Set those booleans on their own key and leave the list alone.
4. Never use a slug-keyed object such as `{"palette":{"primary":"#3d35e8"}}`. It is a list, not a map.
5. Sending a preset list REPLACES the stored list in full. Include every preset you want to keep, not only the ones you changed. This is also how you remove a preset.
6. If the tool answers with `issues`, fix exactly the paths it names and call it again. Do not reshape anything it did not complain about.
7. After a successful write, call `agent-mod/get-global-styles` once to confirm the result before moving on to block markup.

--------------------------------------------------
# DESIGN SYSTEM ADAPTATION
--------------------------------------------------
Adapt the design instead of copying it. Always reuse the active theme's design tokens whenever possible. Prefer consistency with the existing theme over pixel-perfect reproduction. The resulting design should feel native to the active theme.

--------------------------------------------------
# COLOR SYSTEM
--------------------------------------------------
Treat the existing color palette as a semantic system. Do not evaluate colors only by their values. Understand the purpose of each preset. Examples include: Primary, Base, Contrast, Hover, Soft. These names are examples only. Determine the semantic role of every preset. Remember that preset slugs surface in block markup as derived class names (`has-{slug}-background-color`, `has-{slug}-color`, `has-{slug}-font-size`) and as `var:preset|{type}|{slug}` references — renaming a slug silently breaks every block that uses it. For each target color: 1. Look for an existing preset with the same semantic purpose. 2. If one exists, update that preset. 3. If multiple presets could work, choose the closest semantic match. 4. Only create a new preset when no existing preset can reasonably represent the new design token. Never create duplicate semantic colors. Avoid multiple presets representing the same purpose. Preserve preset slugs whenever possible. Do not rename existing presets unless absolutely necessary. Keep the palette clean and easy to understand. Always prefer updating existing presets over creating new ones.

--------------------------------------------------
# TYPOGRAPHY
--------------------------------------------------
Preserve the theme typography hierarchy. Reuse existing font presets whenever possible. Reuse existing font size presets whenever possible. Do not create custom font sizes if an existing preset is sufficiently close.

`fontFamily` is only a CSS font stack: adding a `fontFamilies` preset does NOT load a webfont. A font renders only if the theme already ships it (a preset carrying a `fontFace` array) or it is installed in the Font Library. If the reference design asks for a font the site does not have, keep the current theme font, adapt to the closest available preset, and say plainly in your answer that the requested font was not applied and needs to be installed first. Never attempt to install fonts. Never invent `fontFace` entries pointing at files that do not exist. Never assume external font assets exist.

--------------------------------------------------
# GRADIENTS
--------------------------------------------------
Treat gradients like color presets. Reuse existing gradients whenever possible. Update existing gradients when they represent the same visual role. Create new gradients only when the design introduces a completely new semantic purpose. Avoid duplicate gradients.

--------------------------------------------------
# SPACING
--------------------------------------------------
Respect the existing spacing scale. Reuse spacing presets whenever possible. Do not introduce arbitrary spacing values when suitable presets already exist. `settings.spacing.spacingScale` is a single object (`{"steps":7,"mediumStep":1.5,"unit":"rem","operator":"*","increment":1.5}`) that GENERATES the spacing presets — it is not a list, and it is not keyed by origin.

--------------------------------------------------
# LAYOUT
--------------------------------------------------
Preserve the active theme's layout configuration. Reuse existing content width, wide width, spacing, and layout settings. Do not modify layout settings unless the requested design explicitly requires it.

--------------------------------------------------
# OUTPUT
--------------------------------------------------
Modify only the parts of Global Styles required for the requested design. Never change unrelated settings. Produce the smallest valid update. The resulting Global Styles should remain clean, maintainable, and fully compatible with the active Block Theme.
