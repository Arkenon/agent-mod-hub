Title: WordPress Block Markup
Slug: wordpress-block-markup
Description: Use this skill whenever a request involves writing or editing the content of a post, page, template, template part, or block pattern on this site. All of that content is serialized WordPress block markup, and it is written through abilities never by editing theme files or writing PHP.
Version: 1.1.0
===

## The abilities you work with (more may be added, always check existing abilities)

| Goal | Abilities |
|---|---|
| Find content | `agent-mod/list-posts`, `agent-mod/list-recent-posts`, `agent-mod/list-templates`, `agent-mod/list-patterns` |
| Read markup | `agent-mod/get-post`, `agent-mod/get-template`, `agent-mod/get-pattern` |
| Write markup | `agent-mod/create-post`, `agent-mod/update-post`, `agent-mod/add-or-update-template`, `agent-mod/create-pattern`, `agent-mod/update-pattern`, `agent-mod/duplicate-pattern` |
| Design tokens | `agent-mod/get-global-styles`, `agent-mod/update-global-styles` |
| Validate markup | `agent-mod/validate-block-markup` |

Every read ability returns the content in an `html` field. Every write ability takes the
same shape back in its `html` parameter. This is a **round trip**: read modify write.

## Non-negotiable rules

1. **Only block markup is accepted.** The `html` parameter is parsed with `parse_blocks()`
   and anything without a block delimiter is discarded. If you send plain HTML
   (`<h2>Title</h2><p>Text</p>`) the ability fails with *"No valid blocks found in the
   provided block markup."* Every top-level element must be wrapped in
   `<!-- wp:{block} -->`  `<!-- /wp:{block} -->`.
2. **Writes replace the entire content.** Never send a fragment. Read the current markup
   first, apply your edit to the full document, and send the whole thing back. On a
   template this means keeping the existing `core/template-part` header/footer blocks.
3. **No PHP, no script tags, no style tags, no custom CSS class names.** All design
   goes through block attributes and theme.json preset slugs. Content is stored in the
   database, so there are no `esc_html__()` / text-domain concerns here.
4. **Never invent preset slugs.** Call `agent-mod/get-global-styles` before styling and
   use only the color, font-size, and spacing slugs the theme actually defines. An
   unknown slug renders as no style at all.
5. **Never invent attribute paths.** An attribute the block does not define is silently
   dropped by `save()`  the styling disappears *and* the block fails validation. If you
   are not certain a path exists, read a working example instead of guessing.
6. **Only reference media that exists.** A `core/image` block needs a real attachment
   `id` and `url`; do not invent IDs or hotlink external placeholder images. If no image URL or Id provided, use empty Image Block.
7. **Validate before writing.** Run `agent-mod/validate-block-markup` on the full
   markup before every write ability call. It reports malformed delimiters, invalid
   JSON attribute objects, unregistered block names, and content outside block
   delimiters. Fix every issue and re-validate; never call a write ability with markup
   that did not come back `valid: true`.
8. **Verify after writing.** Re-read with `get-post` / `get-template` / `get-pattern` and
   confirm the markup came back as intended before reporting success.

## Anatomy of a block

```
<!-- wp:heading {"level":3,"fontSize":"x-large","style":{"typography":{"textAlign":"center"}}} -->
<h3 class="wp-block-heading has-text-align-center has-x-large-font-size">Title</h3>
<!-- /wp:heading -->
```

Three parts, and **all three must agree**:

- the opening comment with the block name and a JSON attribute object,
- the saved HTML, which must carry exactly the classes and inline styles those
  attributes imply  no more, no less,
- the matching closing comment.

If the attributes and the HTML disagree, the editor shows *"This block contains
unexpected or invalid content"* and offers block recovery. That is the failure you are
avoiding. Recovery is not a safe net: it regenerates the HTML from the attributes, so
anything you expressed **only** in the HTML is lost, and anything you expressed through
an invented attribute path is lost too.

### Delimiter syntax

- Core blocks drop the namespace: `wp:paragraph`, not `wp:core/paragraph`. Third-party
  blocks keep theirs: `wp:woocommerce/product-price`.
- Blocks that save no HTML of their own are self-closing  the slash goes before the
  `-->` and there is no closing comment: `<!-- wp:site-title /-->`,
  `<!-- wp:post-content /-->`, `<!-- wp:template-part {"slug":"header"} /-->`. Blocks
  that do save HTML (paragraph, heading, group, spacer, separator, ) always need the
  closing comment.
- The attribute object must be valid, single-line JSON: double-quoted keys and strings,
  no trailing commas, no comments, no single quotes. Omit it entirely when empty 
  write `<!-- wp:paragraph -->`, not `<!-- wp:paragraph {} -->`.
- Only non-default attributes are serialized. Do not write `{"level":2}` on a heading
  (2 is the default) or `{"dropCap":false}`.
- Do not add attributes the parent context makes meaningless  e.g. a child's
  `{"layout":{"selfStretch":"fit","flexSize":null}}` when the parent layout is
  `constrained`.
- Nesting must be balanced and correctly ordered; inner blocks live inside the parent's
  saved HTML element, not between the parent's comments and its `<div>`. Close every
  HTML tag you open  a single missing `</div>` invalidates the whole parent chain.
- Text alignment on headings and paragraphs is stored as
  `{"style":{"typography":{"textAlign":"center"}}}` plus the `has-text-align-center`
  class. Never use the legacy top-level `"align"` or `"textAlign"` attributes for text
  alignment  current WordPress versions migrate them through block recovery.

## Attribute class/style mapping

**This table is the single biggest source of invalid blocks.** Every style attribute
below emits *both* an inline style **and** a class. Omitting the class is the most common
failure, and it is silent  the block simply refuses to validate.

### Custom (raw) values

| Attribute | Required class | Required inline style |
|---|---|---|
| `"style":{"color":{"text":"#a0a0c0"}}` | `has-text-color` | `color:#a0a0c0` |
| `"style":{"color":{"background":"#111111"}}` | `has-background` | `background-color:#111111` |
| `"style":{"color":{"gradient":"linear-gradient()"}}` | `has-background` | `background:linear-gradient()` |
| `"style":{"border":{"color":"rgba(255,255,255,0.1)"}}` | `has-border-color` | `border-color:rgba(255,255,255,0.1)` |
| `"style":{"border":{"color":"var:preset--color--border"}}` | `has-border-color` | `border-color:var(--wp--preset--color--border` |
| `"style":{"border":{"width":"1px"}}` |  | `border-width:1px` |
| `"style":{"border":{"style":"solid"}}` |  | `border-style:solid` |
| `"style":{"typography":{"textAlign":"center"}}` | `has-text-align-center` |  |
| `"align":"wide"` | `alignwide` |  |
| `"align":"full"` | `alignfull` |  |

The `has-border-color` class is the most frequently forgotten one: whenever
`style.border.color` (or a `borderColor` preset) is set, the element MUST carry
`has-border-color`.


## Most-Used Blocks in Patterns

Preset slugs in these examples (`base`, `contrast`, `primary`, `soft`, `border`,
spacing numbers) are **placeholders**  every theme defines its own. Always
substitute the active theme's actual slugs from `agent-mod/get-global-styles`.

### Design Blocks

**Tabs:**
```html
<!-- wp:tabs -->
<div class="wp-block-tabs">
  <!-- wp:tab-list -->
  <div role="tablist" class="wp-block-tab-list">
    <button type="button" role="tab">Tab 1</button>
    <button type="button" role="tab">Tab 2</button>
  </div>
  <!-- /wp:tab-list -->

  <!-- wp:tab-panels -->
  <div class="wp-block-tab-panels">
    <!-- wp:tab-panel {"label":"Tab 1"} -->
    <section role="tabpanel" tabindex="0" class="wp-block-tab-panel">
      <!-- wp:paragraph -->
      <p>Panel 1</p>
      <!-- /wp:paragraph -->
    </section>
    <!-- /wp:tab-panel -->

    <!-- wp:tab-panel {"label":"Tab 2"} -->
    <section role="tabpanel" tabindex="0" class="wp-block-tab-panel">
      <!-- wp:paragraph -->
      <p>Panel 2</p>
      <!-- /wp:paragraph -->
    </section>
    <!-- /wp:tab-panel -->
  </div>
  <!-- /wp:tab-panels -->
</div>
<!-- /wp:tabs -->
```

**Accordion:**
```html
<!-- wp:accordion -->
<div role="group" class="wp-block-accordion">
  <!-- wp:accordion-item -->
  <div class="wp-block-accordion-item">
    <!-- wp:accordion-heading -->
    <h3 class="wp-block-accordion-heading has-icon has-icon-right">
      <button type="button" class="wp-block-accordion-heading__toggle">
        <span class="wp-block-accordion-heading__toggle-title">Accordion Title</span>
        <span class="wp-block-accordion-heading__toggle-icon" aria-hidden="true">+</span>
      </button>
    </h3>
    <!-- /wp:accordion-heading -->
    <!-- wp:accordion-panel -->
    <div role="region" class="wp-block-accordion-panel">
      <!-- wp:paragraph -->
      <p>Accordion Panel Content</p>
      <!-- /wp:paragraph -->
    </div>
    <!-- /wp:accordion-panel -->
  </div>
  <!-- /wp:accordion-item -->
</div>
<!-- /wp:accordion -->
```

### Icon Block
Sample markup is here.

```html
<!-- wp:icon {"icon":"core/settings","style":{"color":{"background":"#e6f7ff"},"dimensions":{"width":"45px"},"spacing":{"padding":{"top":"var:preset|spacing|20","bottom":"var:preset|spacing|20","left":"var:preset|spacing|20","right":"var:preset|spacing|20"},"margin":{"top":"var:preset|spacing|20","bottom":"var:preset|spacing|20","left":"var:preset|spacing|20","right":"var:preset|spacing|20"}},"border":{"radius":"10px","width":"1px"}},"textColor":"accent-1","borderColor":"accent-3"} /-->
```

**Avaliable Icons**: core/arrow-down-left, core/arrow-down-right, core/arrow-down, core/arrow-left, core/arrow-right, core/arrow-up-left, core/arrow-up-right, core/arrow-up, core/at-symbol, core/audio, core/bell, core/block-default, core/block-meta, core/block-table, core/calendar, core/capture-photo, core/capture-video, core/cart, core/category, core/caution, core/chart-bar, core/check, core/chevron-down, core/chevron-down-small, core/chevron-left, core/chevron-left-small, core/chevron-right, core/chevron-right-small, core/chevron-up, core/chevron-up-down, core/chevron-up-small, core/comment, core/cover, core/create, core/desktop, core/download, core/drawer-left, core/drawer-right, core/envelope, core/error, core/external, core/file, core/gallery, core/group, core/heading, core/help, core/home, core/image, core/info, core/key, core/language, core/map-marker, core/menu, core/mobile, core/more-horizontal, core/more-vertical, core/next, core/paragraph, core/payment, core/pencil, core/people, core/plus, core/plus-circle, core/previous, core/published, core/quote, core/receipt, core/rss, core/scheduled, core/search, core/settings, core/shadow, core/share, core/shield, core/shuffle, core/star-empty, core/star-filled, core/star-half, core/store, core/styles, core/symbol, core/symbol-filled, core/table, core/tablet, core/tag, core/tip, core/upload, core/verse

### Layout Blocks

**Group**  primary container, supports all layout types:
```html
<!-- wp:group {"layout":{"type":"constrained"}} -->
<div class="wp-block-group">
  <!-- inner blocks -->
</div>
<!-- /wp:group -->
```

Layout types:
- `{"type":"constrained"}`  centered with max-width (default for sections)
- `{"type":"constrained","contentSize":"800px","wideSize":"1200px"}`  custom widths
- `{"type":"flex","flexWrap":"nowrap"}`  horizontal row
- `{"type":"flex","orientation":"vertical"}`  vertical stack
- `{"type":"grid","columnCount":3}`  CSS grid with fixed columns
- `{"type":"grid","minimumColumnWidth":"250px"}`  responsive auto-fill grid

Tag name override: `{"tagName":"section"}`, `{"tagName":"header"}`, `{"tagName":"footer"}` Default is div

**Card (bordered group)**  note `has-border-color` and the inline declaration order
`border-color; border-style; border-width; border-radius`:
```html
<!-- wp:group {"style":{"border":{"color":"var:preset|color|border","width":"1px","style":"solid","radius":"10px"},"spacing":{"padding":{"top":"var:preset|spacing|40","right":"var:preset|spacing|30","bottom":"var:preset|spacing|40","left":"var:preset|spacing|30"},"blockGap":"var:preset|spacing|20"}},"backgroundColor":"soft","layout":{"type":"constrained"}} -->
<div class="wp-block-group has-border-color has-soft-background-color has-background" style="border-color:var(--wp--preset--color--border);border-style:solid;border-width:1px;border-radius:10px;padding-top:var(--wp--preset--spacing--40);padding-right:var(--wp--preset--spacing--30);padding-bottom:var(--wp--preset--spacing--40);padding-left:var(--wp--preset--spacing--30)">
  <!-- inner blocks -->
</div>
<!-- /wp:group -->
```

**Columns / Column:**
```html
<!-- wp:columns -->
<div class="wp-block-columns">
  <!-- wp:column {"width":"66.66%"} -->
  <div class="wp-block-column" style="flex-basis:66.66%">
    <!-- inner blocks -->
  </div>
  <!-- /wp:column -->
  <!-- wp:column {"width":"33.33%"} -->
  <div class="wp-block-column" style="flex-basis:33.33%">
    <!-- inner blocks -->
  </div>
  <!-- /wp:column -->
</div>
<!-- /wp:columns -->
```

### Content Blocks

**Heading:**
```html
<!-- wp:heading {"level":2,"fontSize":"x-large"} -->
<h2 class="wp-block-heading has-x-large-font-size">Heading text</h2>
<!-- /wp:heading -->
```

**Paragraph:**
```html
<!-- wp:paragraph {"fontSize":"medium","textColor":"contrast"} -->
<p class="has-contrast-color has-text-color has-medium-font-size">Body text</p>
<!-- /wp:paragraph -->
```

**Image:**
```html
<!-- wp:image {"sizeSlug":"full","linkDestination":"none"} -->
<figure class="wp-block-image size-full">
  <img src="https://example.com/image.jpg" alt="Descriptive alt text" />
</figure>
<!-- /wp:image -->
```

**Cover:**
```html
<!-- wp:cover {"url":"https://example.com/bg.jpg","dimRatio":60,"overlayColor":"contrast","minHeight":500,"minHeightUnit":"px","layout":{"type":"constrained"}} -->
<div class="wp-block-cover" style="min-height:500px">
  <span aria-hidden="true" class="wp-block-cover__background has-contrast-background-color has-background-dim-60 has-background-dim"></span>
  <img class="wp-block-cover__image-background" alt="" src="https://example.com/bg.jpg" data-object-fit="cover" />
  <div class="wp-block-cover__inner-container">
    <!-- inner blocks -->
  </div>
</div>
<!-- /wp:cover -->
```

**Buttons / Button:**
```html
<!-- wp:buttons {"layout":{"type":"flex","justifyContent":"center"}} -->
<div class="wp-block-buttons">
  <!-- wp:button {"backgroundColor":"primary","textColor":"base"} -->
  <div class="wp-block-button"><a class="wp-block-button__link has-base-color has-primary-background-color has-text-color has-background wp-element-button">Click me</a></div>
  <!-- /wp:button -->
  <!-- wp:button {"className":"is-style-outline"} -->
  <div class="wp-block-button is-style-outline"><a class="wp-block-button__link wp-element-button">Secondary</a></div>
  <!-- /wp:button -->
</div>
<!-- /wp:buttons -->
```

Button pitfalls  the most recovery-prone block:
- The wrapper `<div class="wp-block-button">` stays bare (plus `is-style-*` when set);
  ALL color/size classes go on the inner `<a>`, never on the wrapper.
- The `<a>` always needs both `wp-block-button__link` and `wp-element-button`.
- Each `wp:button` lives inside a `wp:buttons` parent; the `wp:buttons` div contains
  only the button children, no other markup.
- Custom width: `{"width":50}` wrapper classes `has-custom-width wp-block-button__width-50`.

**Spacer:**
```html
<!-- wp:spacer {"height":"var:preset|spacing|50"} -->
<div style="height:var(--wp--preset--spacing--50)" aria-hidden="true" class="wp-block-spacer"></div>
<!-- /wp:spacer -->
```

**Separator:**
```html
<!-- wp:separator {"backgroundColor":"contrast","className":"is-style-wide"} -->
<hr class="wp-block-separator has-text-color has-contrast-color has-alpha-channel-opacity has-contrast-background-color has-background is-style-wide" />
<!-- /wp:separator -->
```

**Media & Text:**
```html
<!-- wp:media-text {"mediaPosition":"right","mediaType":"image","mediaWidth":40} -->
<div class="wp-block-media-text has-media-on-the-right is-stacked-on-mobile" style="grid-template-columns:auto 40%">
  <div class="wp-block-media-text__content">
    <!-- inner blocks -->
  </div>
  <figure class="wp-block-media-text__media">
    <img src="https://example.com/image.jpg" alt="Description" />
  </figure>
</div>
<!-- /wp:media-text -->
```

**Query Loop (post listing):**
```html
<!-- wp:query {"queryId":0,"query":{"perPage":3,"pages":0,"offset":0,"postType":"post","order":"desc","orderBy":"date","inherit":false}} -->
<div class="wp-block-query">
  <!-- wp:post-template {"layout":{"type":"grid","columnCount":3}} -->
    <!-- wp:post-featured-image {"isLink":true} /-->
    <!-- wp:post-title {"isLink":true,"fontSize":"large"} /-->
    <!-- wp:post-excerpt {"excerptLength":20} /-->
  <!-- /wp:post-template -->
</div>
<!-- /wp:query -->
```

## Style Attribute Structure

The `style` attribute holds custom values (not preset slugs):

```json
{
  "style": {
    "spacing": {
      "padding": {"top":"var:preset|spacing|50","right":"var:preset|spacing|50","bottom":"var:preset|spacing|50","left":"var:preset|spacing|50"},
      "margin": {"top":"0","bottom":"0"},
      "blockGap": "var:preset|spacing|30"
    },
    "border": {
      "radius": "8px",
      "width": "1px",
      "color": "var:preset|color|contrast",
      "style": "solid"
    },
    "color": {
      "background": "#1a1a2e",
      "text": "#ffffff",
      "gradient": "linear-gradient(135deg,rgb(6,147,227) 0%,rgb(155,81,224) 100%)"
    },
    "typography": {
      "fontSize": "clamp(1rem, 2vw, 1.5rem)",
      "lineHeight": "1.4",
      "letterSpacing": "-0.02em"
    }
  }
}
```

Preset reference syntax in style values: `var:preset|{type}|{slug}` (not CSS `var()`)

## Preset Class Naming Convention

When using preset slugs (not inline style), blocks get CSS classes:
- `"backgroundColor":"primary"` `has-primary-background-color has-background`
- `"textColor":"contrast"` `has-contrast-color has-text-color`
- `"fontSize":"large"` `has-large-font-size`
- `"fontFamily":"heading"` `has-heading-font-family`
- `"gradient":"vivid-cyan-blue-to-vivid-purple"` `has-vivid-cyan-blue-to-vivid-purple-gradient-background has-background`

## Block Locking

Prevent users from modifying pattern structure:

```json
{
  "lock": {"move": true, "remove": true}
}
```

On container blocks, `templateLock` constrains children:
- `"templateLock":"all"`  no insert, move, or remove
- `"templateLock":"insert"`  no adding/removing, can move
- `"templateLock":"contentOnly"`  only text/media editable, structure locked

## Block Alignment & Width

The `align` attribute and its class must appear together:
- `{"align":"wide"}` class `alignwide` on the root element (spans `wideSize`)
- `{"align":"full"}` class `alignfull` (spans the viewport edge)

Width strategy  decide deliberately, never leave it to chance:
1. Read `settings.layout.contentSize` and `settings.layout.wideSize` from
   `agent-mod/get-global-styles` before composing a section.
2. Typical section: outer group `{"align":"full","layout":{"type":"constrained"}}`
   with background and vertical padding.
3. Inner content that the design shows wider than the text column  card grids,
   image rows, multi-column features  gets `{"align":"wide"}` (class `alignwide`).
   Most themes use the wide width for these; default content width is for prose
   (headings, paragraphs) only.
