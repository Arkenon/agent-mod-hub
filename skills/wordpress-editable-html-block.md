Title: WordPress Editable HTML Block Composition
Slug: wordpress-editable-html-block
Description: Use this skill when generating raw WordPress Custom HTML block (`core/html`) markup by hand — i.e. writing the serialized `<!-- wp:html -->...<!-- /wp:html -->` block markup directly as code output, not by using the block editor UI. The output mixes static, designer-controlled HTML/CSS/JS with a small number of editable inner blocks (mainly Image, Heading, Paragraph) so content can be swapped without touching layout. Requires WordPress 7.1+ / Gutenberg's `innerContent` block support (gutenberg#79115).
Version: 1.0.0
===

WordPress 7.1 lets a Custom HTML block interleave static HTML with regular,
editable inner blocks in place, instead of forcing an all-static-or-all-blocks
choice. This skill is about **generating that serialized markup directly as
code** — there is no block editor, no "Edit HTML" modal, no clicking through
the UI. The agent's job is to produce a complete, valid
`<!-- wp:html -->...<!-- /wp:html -->` string that WordPress can parse,
store as `post_content`, and render correctly on both the editor canvas and
the front end.

# Mental model

- The markup you generate is the single source of truth. Static fragments
  and `<!-- wp:... -->`-delimited blocks are interleaved at arbitrary depth
  inside one `<!-- wp:html -->` block.
- Inner blocks you embed become editable in the block editor afterward, but
  are **locked**: no moving, no removing, no adding siblings, no alignment
  controls. The surrounding static structure never changes shape. This is
  the point — a human editor can only ever change text/media content, never
  break the layout.
- Serialization is byte-exact: whatever you write is exactly what gets
  stored and re-parsed. There is no server-side reformatting to rely on or
  guard against, since you are producing the final markup yourself, not
  typing into a UI that reformats it.
- Static HTML (including any `<script>`/`<style>` you embed) renders
  **inert** in the block editor's canvas: WordPress assigns it via
  `innerHTML`, so inline `on*` handlers are stripped and `<script>` tags
  never execute there. This is normal DOM behavior (scripts inserted via
  `innerHTML` never run), not a bug — do not "fix" it. Scripts only ever
  execute on the real front-end page load.

# Which blocks to use for editable slots

Default to these three — they cover "change the text / change the picture"
without any risk of the editor accidentally restructuring the design:

- **`core/paragraph`** — body text.
- **`core/heading`** — titles/subtitles, any level.
- **`core/image`** — a single image, including alt text.

Technically, **any block that has valid save markup can be embedded** the
same way (list, quote, button, columns, custom blocks, etc.) — the parser
just recognizes the `<!-- wp:blockname -->` delimiters wherever they occur,
there is no fixed allowlist. But prefer restraint: this pattern exists to
let content be edited without disturbing a hand-built design, so avoid
embedding structural/layout blocks (`core/cover`, `core/columns`,
`core/group`, `core/media-text`, etc.) inside a Custom HTML block — those
carry their own wrapper markup, padding, and layout assumptions that fight
with the static shell around them and defeat the purpose. If a design needs
more than text/heading/image edits, that's a sign it should be a real block
pattern, not a Custom HTML block.

## Reference snippets (use the exact save markup + wrapper classes)

Paragraph:
```html
<!-- wp:paragraph -->
<p>Editable paragraph text.</p>
<!-- /wp:paragraph -->
```

Heading (level is required in save markup even though optional in JSON):
```html
<!-- wp:heading {"level":2} -->
<h2 class="wp-block-heading">Editable heading</h2>
<!-- /wp:heading -->
```

Image:
```html
<!-- wp:image {"sizeSlug":"large"} -->
<figure class="wp-block-image size-large"><img src="https://example.com/image.jpg" alt="Descriptive alt text"/></figure>
<!-- /wp:image -->
```

The parser does not inject wrapper classes (`wp-block-heading`,
`wp-block-image size-*`) for you — write them explicitly or the block will
parse as invalid/un-styled.

# Workflow — follow these steps in order

1. **Design the static shell as plain HTML.** Container elements, classes,
   `<style>` — write it exactly as it should render, with no blocks yet.
   Scope IDs/classes so they won't collide if the same markup could ever
   appear more than once on a page (prefix with a unique string, or derive
   IDs from a per-instance value if you know one is available).

2. **Drop in editable slots using the reference snippets above**, placed at
   the exact position they should render inside the static structure. Keep
   the set of embedded blocks to paragraph/heading/image unless the task
   explicitly calls for something else.

3. **Write any JS defensively — never assume execution timing or position.**
   Wrap all DOM access in `DOMContentLoaded` so it doesn't matter where in
   the document the script physically sits, and null-guard every lookup so
   a missing/duplicated element fails silently instead of throwing:

   ```html
   <script>
   document.addEventListener('DOMContentLoaded', function () {
     var btn = document.getElementById('likeBtn');
     if (!btn) return; // guard: element may be missing or duplicated
     btn.addEventListener('click', function () {
       // ...
     });
   });
   </script>
   ```

4. **Assemble the full output**, wrapped once in
   `<!-- wp:html -->` / `<!-- /wp:html -->`:

   ```html
   <!-- wp:html -->
   <style>...</style>
   <div class="...">
     <!-- wp:image {"sizeSlug":"large"} -->
     <figure class="wp-block-image size-large"><img src="..." alt="..."/></figure>
     <!-- /wp:image -->
     <!-- wp:heading {"level":2} -->
     <h2 class="wp-block-heading">...</h2>
     <!-- /wp:heading -->
     <!-- wp:paragraph -->
     <p>...</p>
     <!-- /wp:paragraph -->
   </div>
   <script>...</script>
   <!-- /wp:html -->
   ```

5. **State the testing caveat explicitly when handing off the output.**
   CSS and the locked/editable structure are visible in the block editor
   canvas; embedded `<script>` never executes there. Tell whoever inserts
   this markup to verify JS on the actual published/previewed front end,
   not by looking at the editor.

# Output checklist

- [ ] Exactly one `<!-- wp:html -->` / `<!-- /wp:html -->` wrapper around
      the whole thing.
- [ ] Only paragraph/heading/image used as editable slots, unless the task
      explicitly asked for something else — no layout blocks (cover,
      columns, group, media-text) nested inside.
- [ ] Every embedded block's save markup includes its required wrapper
      class(es).
- [ ] All JS wrapped in `DOMContentLoaded`, every `getElementById`/query
      result null-checked before use.
- [ ] IDs/classes scoped to avoid collisions if the block can repeat on a
      page.
- [ ] Output includes a one-line note that JS must be verified on the
      front end, not the editor canvas.
