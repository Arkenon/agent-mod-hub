**Title:** WordPress PHP-Only Block Registration

**Slug:** wordpress-php-based-blocks

**Description:** Use this skill whenever a request asks for a new Gutenberg block to be built in PHP only — no JavaScript, no build step, no `block.json`/`registerBlockType()` — for a simple, server-rendered, non-interactive block. Covers the `autoRegister` support flag added to `register_block_type()`, its attribute rules, auto-generated Inspector Controls, and its hard limitations versus a normal JS-registered block.

**Version:** 1.0.0

===

## What this is

WordPress added a way to register a block **entirely from PHP**, with no `block.json`, no JS entry point, no React, no build tooling. You call the existing `register_block_type()` function and set the `autoRegister` support flag. WordPress then:

- Exposes the block in the block inserter automatically.
- Auto-generates Inspector Controls (sidebar UI) for the block's attributes, based on their declared `type`.
- Renders the block preview in the editor via a server round-trip (ServerSideRender-style): every attribute change re-calls the PHP `render_callback` over the REST API and refreshes the preview.

This is **only** for simple, server-rendered, low-interactivity blocks. It is not a replacement for the client-side block API and is deliberately not as featureful. Reach for it when a request explicitly wants "no JS block" / "PHP-only block" / a block that just outputs server-rendered markup with a few editable settings (text, numbers, toggles, a small enum/select).

Requires a recent WordPress core version that ships this feature (introduced as a Gutenberg/core dev feature, referenced under core ticket **#64639**). If the target site's WordPress version is unknown or old, verify `register_block_type()` accepts an `autoRegister` supports flag before relying on it — there is no runtime feature-detection function, so this has to be a version check against the changelog, not a capability probe.

## Minimal working example

```php
function gutenberg_register_php_only_blocks() {
    register_block_type(
        'my-plugin/example',
        array(
            'title'           => __( 'My Example Block', 'myplugin' ),
            'attributes'      => array(
                'title'   => array(
                    'label'   => __( 'Title', 'myplugin' ),
                    'type'    => 'string',
                    'default' => 'Hello World',
                ),
                'count'   => array(
                    'label'   => __( 'Count', 'myplugin' ),
                    'type'    => 'integer',
                    'default' => 5,
                ),
                'enabled' => array(
                    'label'   => __( 'Enabled?', 'myplugin' ),
                    'type'    => 'boolean',
                    'default' => true,
                ),
                'size'    => array(
                    'label'   => __( 'Size', 'myplugin' ),
                    'type'    => 'string',
                    'enum'    => array( 'small', 'medium', 'large' ),
                    'default' => 'medium',
                ),
            ),
            'render_callback' => function ( $attributes ) {
                return sprintf(
                    __( '<p>%s: %d items (%s)</p>', 'myplugin' ),
                    esc_html( $attributes['title'] ),
                    $attributes['count'],
                    $attributes['size']
                );
            },
            'supports'        => array(
                'autoRegister' => true,
            ),
        )
    );
}

add_action( 'init', 'gutenberg_register_php_only_blocks' );
```

That's the whole block: no `block.json`, no `index.js`, no build step, no separate registration on the JS side. `render_callback` receives the resolved `$attributes` array (already defaulted/validated) and returns the HTML string to output on both the frontend and the editor preview.

## Attribute rules

- **Supported `type` values:** `string`, `integer`, `number`, `boolean`. Nothing else auto-generates a control.
- Give every attribute a `label` (wrap it in `__()`) — this is what the sidebar control is titled. There is no separate JS control definition to write.
- `enum` works for `string` attributes and renders as a select-style control, but the **values are shown as-is** — there is no way to map an enum value to a translated/custom display label.
- Attributes must **not** be sourced from HTML (`source`, `selector`, etc.) and must not use the `local` role. The editor auto-control generator silently skips (does not render a control for) any attribute with `role: 'local'` or with an unsupported `type` — the block still registers, but that attribute becomes uneditable from the sidebar.
- Attributes serialize the normal way: as JSON in the block comment delimiter. There is no `save()` to hand-author markup with data attributes.

## Hard limitations — do not attempt these with autoRegister blocks

1. **No inner blocks / no nesting.** A PHP-only block cannot act as a container for child blocks.
2. **No `save()` function.** Only `render_callback` exists. You cannot persist content into `post_meta` from a custom save step or parse saved HTML back into attributes — the block is 100% dynamic/server-rendered, every time.
3. **No media/file upload attributes, no rich text, no textarea.** If the block needs an image picker, a WYSIWYG field, or multi-line text editing, this API cannot express it — build a normal JS block instead.
4. **`uses_context` is unreliable across the REST preview round-trip.** Context values are available on the *initial* render, but may not be present when the editor re-renders the preview via REST after an attribute change. Don't depend on block context for anything the preview must reflect live.
5. **Don't name an attribute `style`.** It collides with WordPress's implicit style object used for color/spacing/typography supports.

## Native block supports still work

Because this still goes through `register_block_type()`, you can declare the normal `supports` entries (`align`, `color`, `typography`, `spacing`, `border`, etc.) alongside `autoRegister => true`. Inside `render_callback`, call `get_block_wrapper_attributes()` to get the classes/inline styles those supports generate, and apply them to your wrapper element yourself — nothing does this for you automatically the way `useBlockProps()` does on the JS side.

```php
'render_callback' => function ( $attributes ) {
    $wrapper_attributes = get_block_wrapper_attributes();
    return sprintf(
        '<div %1$s><p>%2$s</p></div>',
        $wrapper_attributes,
        esc_html( $attributes['title'] )
    );
},
```

## Editor assets and frontend scripts

- If the block needs its own editor-only CSS, enqueue it via the `enqueue_block_editor_assets` hook — `wp_enqueue_block_style()` does not fire for autoRegistered blocks the way it does for a `block.json`-declared block.
- If the block needs frontend JavaScript (e.g. a small behavior that doesn't require full interactivity), register the script at `init` as usual, but **enqueue it from inside `render_callback`** rather than unconditionally — this naturally deduplicates the enqueue when the block appears multiple times on a page, since `wp_enqueue_script()` is idempotent per handle.

## When to reach for the normal JS block API instead

If the request needs any of: inner blocks, a save-to-markup block (not fully dynamic), rich text editing, media/image controls, textarea/long-form text, or reliable use of block context in the editor preview — this API is the wrong tool. Fall back to a standard `block.json` + `registerBlockType()` block.
