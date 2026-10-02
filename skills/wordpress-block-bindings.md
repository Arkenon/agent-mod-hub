**Title:** WordPress Block Bindings

**Slug:** wordpress-block-bindings

**Description:** Use this skill whenever content or templates should show dynamic data (post meta, post date/link, term data, pattern overrides, or a custom data source) instead of static text — by writing `metadata.bindings` into block markup. Covers which blocks/attributes can be bound, the exact markup for every built-in source, how to expose post meta, and when a PHP Code Snippet is required for a custom source or value formatting.

**Version:** 1.1.0

===

## Writing code: use Code Snippets

You run inside WordPress, not in an IDE or terminal — you cannot edit theme or plugin files. Whenever this skill calls for code (PHP, JavaScript or CSS), deliver it as a Code Snippet using the `agent-mod-pro` abilities: `get-snippets` / `get-snippet` to look for an existing one first, `validate-snippet` to check the code, `add-snippet` to create it and `update-snippet` to change it (plus `set-snippet-status`, `remove-snippet` and the code-reading abilities when needed). Prefer updating an existing snippet over creating a duplicate. A new snippet is a reviewable draft: tell the user it takes effect only once it is activated. Explain what the code must do, then create the snippet — don't paste code into chat and ask the user to place it somewhere.

## What this is

Block Bindings connect a block **attribute** to a data **source**. The block stores a *reference* to the source, not the value. At render time WordPress resolves the reference and replaces the attribute's value; when the source changes, every bound block updates automatically. (Requires WordPress 6.5+; `core/post-data`, `core/term-data` and extra supported attributes need 6.9+.)

You work inside WordPress, not an IDE. Your main tool is **block markup**: you add bindings by writing `metadata.bindings` in the block comment delimiter. Anything that needs code is delivered as a **Code Snippet** (see "When code is needed").

## Markup syntax

Bindings live in the block's `metadata` attribute:

```
metadata.bindings → { attribute_name → { "source": "...", "args": { ... } } }
```

```html
<!-- wp:paragraph {"metadata":{"bindings":{"content":{"source":"core/post-meta","args":{"key":"my_custom_field"}}}}} -->
<p>Fallback text</p>
<!-- /wp:paragraph -->
```

Rules:

- The key under `bindings` is the **attribute name** being bound (`content`, `url`, `alt`, …), not a CSS selector.
- `source` is the source name; `args` is the source-specific configuration (omit when the source takes none).
- Keep the block's normal inner HTML as **fallback / editor placeholder** (the `<p>`, `<h2>`, `<img>`, `<a>` etc. must still be valid for that block). The bound attribute overrides it on render. An empty `<p></p>` is fine for text.
- JSON must be valid and the block comment must stay on valid single-block syntax; remember `/` in source names needs no escaping in JSON.
- A bound attribute is dynamic: don't also hand-type the "real" value expecting it to persist — the source wins on the frontend.

## Which blocks and attributes can be bound

| Block | Bindable attributes |
|-------|--------------------|
| `core/paragraph` | `content` |
| `core/heading` | `content` |
| `core/button` | `url`, `text`, `linkTarget`, `rel` |
| `core/image` | `id`, `url`, `title`, `alt`, `caption` |
| `core/navigation-link` | `url` |
| `core/navigation-submenu` | `url` |
| `core/post-date` | `datetime` |

Anything not listed (e.g. `core/group`, `core/cover` background, `core/list`, `core/quote`) **cannot** be bound by markup alone. On WordPress 6.9+ the list can be extended with the `block_bindings_supported_attributes_{block_name}` filter — that is PHP, so it goes through a Code Snippet. If a user wants an unsupported block/attribute dynamic, say so and suggest a supported block (e.g. a Button for a dynamic link) or a different approach instead of writing a binding that will silently do nothing.

## Built-in sources

### `core/post-meta`

Binds an attribute to a post meta value. Args: `{"key": "meta_key"}`.

```html
<!-- wp:heading {"level":3,"metadata":{"bindings":{"content":{"source":"core/post-meta","args":{"key":"movie_director"}}}}} -->
<h3 class="wp-block-heading"></h3>
<!-- /wp:heading -->
```

Button whose link and label both come from meta:

```html
<!-- wp:button {"metadata":{"bindings":{"url":{"source":"core/post-meta","args":{"key":"ticket_url"}},"text":{"source":"core/post-meta","args":{"key":"ticket_label"}}}}} -->
<div class="wp-block-button"><a class="wp-block-button__link wp-element-button">Buy tickets</a></div>
<!-- /wp:button -->
```

Image with several attributes bound:

```html
<!-- wp:image {"metadata":{"bindings":{"url":{"source":"core/post-meta","args":{"key":"poster_url"}},"alt":{"source":"core/post-meta","args":{"key":"poster_alt"}}}}} -->
<figure class="wp-block-image"><img alt=""/></figure>
<!-- /wp:image -->
```

**The meta must be exposed first.** The binding only works if the post meta is registered with `show_in_rest => true` and `single => true`, and the key must **not** start with an underscore (protected keys are inaccessible). Registration is PHP → **Code Snippet** (see below). Before writing a `core/post-meta` binding, check whether the meta key is already registered (by the theme, a plugin like ACF/Meta Box exposing it to REST, or an earlier snippet). If it isn't, say the binding will render empty until a registration snippet exists, and offer to create it.

Context: `core/post-meta` reads the meta of the **current post** (post context). It works inside a post/page, and inside Query Loop → Post Template items (each item has its own post context). Inside a plain template with no post context there is nothing to read.

### `core/post-data` (WordPress 6.9+)

Reads core post fields. Args: `{"field": "date" | "modified" | "link"}`.

```html
<!-- wp:paragraph {"metadata":{"bindings":{"content":{"source":"core/post-data","args":{"field":"modified"}}}}} -->
<p></p>
<!-- /wp:paragraph -->
```

Useful for a Button whose `url` is the post's `link`, or a `core/post-date` `datetime`.

### `core/term-data` (WordPress 6.9+)

Reads taxonomy term data. Needs term context (`termId` and `taxonomy`), e.g. inside a Terms Query / term template. Args: `{"field": "..."}` with fields `id`, `name`, `link`, `slug`, `description`, `parent`, `count`.

```html
<!-- wp:button {"metadata":{"bindings":{"url":{"source":"core/term-data","args":{"field":"link"}},"text":{"source":"core/term-data","args":{"field":"name"}}}}} -->
<div class="wp-block-button"><a class="wp-block-button__link wp-element-button">Term</a></div>
<!-- /wp:button -->
```

Without term context it returns nothing — don't use it in a regular post template.

### `core/pattern-overrides`

Lets a **synced pattern** expose specific blocks so each place the pattern is used can override that block's content while the rest stays synced. Two parts:

1. Inside the pattern, each overridable block gets a unique `metadata.name` (the override id, unique within the pattern) and a binding with `"source":"core/pattern-overrides"` (no args):

```html
<!-- wp:paragraph {"metadata":{"name":"custom-heading","bindings":{"content":{"source":"core/pattern-overrides"}}}} -->
<p>Default heading text</p>
<!-- /wp:paragraph -->
```

2. Where the pattern is used, the instance carries the overrides keyed by that name:

```html
<!-- wp:block {"ref":123,"content":{"custom-heading":{"content":"My custom heading text"}}} /-->
```

Only attributes that are bindable (table above) can be overridden. The pattern must be a **synced** pattern (`wp_block`). Overrides are about per-instance content, not data from an external source.

## Custom sources and value formatting

A custom source (e.g. "current user's name", "a value from an external API", "an ACF-like option") is registered on the server with `register_block_bindings_source()`. You then **reference** it from markup exactly like a built-in one:

```html
<!-- wp:paragraph {"metadata":{"bindings":{"content":{"source":"myplugin/site-tagline-upper"}}}} -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:image {"metadata":{"bindings":{"url":{"source":"myplugin/random-image","args":{"key":"hero"}}}}} -->
<figure class="wp-block-image"><img alt=""/></figure>
<!-- /wp:image -->
```

Source names are `namespace/name`. Custom sources must be registered **before** the binding renders — a binding that points to an unregistered source renders its fallback content.

Facts to pass into the snippet you write:

- `register_block_bindings_source( $name, $args )` runs on the `init` hook.
- `$args`: `label` (human-readable), `get_value_callback` (required), `uses_context` (optional array of context keys it needs, e.g. `array( 'postId' )`).
- `get_value_callback( array $source_args, WP_Block $block_instance, string $attribute_name )` — all three params optional. `$source_args` is the markup's `args` object; return the value (string/URL/etc.) or `null` for "nothing" (fallback shows). Read context via `$block_instance->context['postId']`.
- **Escape/sanitize what you return** — it is injected into the block's HTML (`esc_html`/`esc_url` as appropriate to the attribute, or return plain data and let the block escape; never return unsanitized user input).
- `block_bindings_source_value` filter (WordPress 6.7+) — `apply_filters( 'block_bindings_source_value', $value, $source_name, $source_args, $block_instance, $attribute_name )` — lets you reformat or override the value of *any* source (built-in or custom), e.g. format a date, add a prefix, fall back to a default when the value is empty. Always check the `$source_name` first and `return $value` untouched for other sources.
- `block_bindings_supported_attributes_{$block_name}` filter (6.9+) — extend or restrict which attributes of a block are bindable (e.g. add `linkDestination` to `core/image`, or return `array()` to disable bindings for a block).
- Capability `edit_block_binding` controls who may create/edit bindings in the editor (admins and editors by default).

Editor-side JavaScript (`registerBlockBindingsSource`, `getValues`, `getFieldsList`) only improves the editor *preview* and the binding picker UI. The frontend works with the PHP registration alone, so don't add a JS part by default. If the user wants a live editor preview, create it as a JavaScript snippet loaded in the editor (`enqueue_block_editor_assets`), using vanilla JS and the global `wp.*` packages (no build step).

## When code is needed → create a Code Snippet

You do not edit theme/plugin files. Whenever one of the following is required, explain it briefly, then **create a function-based Code Snippet** with the `agent-mod-pro` snippet abilities (see "Writing code" above; a snippet is a reviewable draft, activation is the user's decision). PHP is the usual case here; a JavaScript part, if genuinely wanted, also goes in a snippet:

| Need | What the snippet does |
|------|----------------------|
| Expose a meta key for `core/post-meta` | `register_post_meta()` (or `register_meta()`) on `init` with `show_in_rest => true`, `single => true`, a `type`, and a `sanitize_callback`; non-underscore key |
| Custom binding source | `register_block_bindings_source()` on `init` with a `get_value_callback` |
| Format/override a binding's value | `add_filter( 'block_bindings_source_value', … )` |
| Make another block/attribute bindable (6.9+) | `add_filter( 'block_bindings_supported_attributes_{block}', … )` |
| Remove binding editing for a role | remove the `edit_block_binding` cap from that role |

After explaining the needed behaviour, the next step is simply: **create a function-based code snippet for it** (never a class), with unique prefixed function names guarded by `function_exists()`, no opening `<?php` tag, and every hook callback as a named function. Then wire the markup to it. Mention that the snippet must be **activated** before the bindings show real values.

Example of what a snippet should contain (function-based, shown for reference — put it in a snippet, never in a file):

```php
if ( ! function_exists( 'agentmod_register_movie_meta' ) ) {
	function agentmod_register_movie_meta() {
		register_post_meta(
			'post',
			'movie_director',
			array(
				'show_in_rest'      => true,
				'single'            => true,
				'type'              => 'string',
				'sanitize_callback' => 'sanitize_text_field',
			)
		);
	}
	add_action( 'init', 'agentmod_register_movie_meta' );
}

if ( ! function_exists( 'agentmod_register_binding_sources' ) ) {
	function agentmod_register_binding_sources() {
		register_block_bindings_source(
			'agentmod/post-year',
			array(
				'label'              => __( 'Post year', 'agentmod' ),
				'get_value_callback' => 'agentmod_binding_post_year',
				'uses_context'       => array( 'postId' ),
			)
		);
	}
	add_action( 'init', 'agentmod_register_binding_sources' );
}

if ( ! function_exists( 'agentmod_binding_post_year' ) ) {
	function agentmod_binding_post_year( $source_args, $block_instance ) {
		$post_id = isset( $block_instance->context['postId'] ) ? (int) $block_instance->context['postId'] : 0;
		return $post_id ? get_the_date( 'Y', $post_id ) : null;
	}
}

if ( ! function_exists( 'agentmod_format_binding_value' ) ) {
	function agentmod_format_binding_value( $value, $source_name ) {
		if ( 'agentmod/post-year' !== $source_name ) {
			return $value;
		}
		return $value ? sprintf( '© %s', $value ) : $value;
	}
	add_filter( 'block_bindings_source_value', 'agentmod_format_binding_value', 10, 2 );
}
```

## Workflow

1. **Pick the source.** Post data already available (date, link, modified) → `core/post-data`. A custom field → `core/post-meta`. Term info → `core/term-data`. Per-instance editable content of a synced pattern → `core/pattern-overrides`. Anything computed/external → custom source (snippet).
2. **Check the block/attribute is bindable** (table above).
3. **Check prerequisites** — meta key registered with REST support? custom source registered? WordPress ≥ 6.9 for `post-data`/`term-data`? If missing, create the Code Snippet first (and tell the user it must be activated).
4. **Write the block markup** with `metadata.bindings` and sensible fallback inner HTML. Use it in posts, patterns, template parts, or templates as the task requires — bindings work anywhere the context exists (post meta / post data need a post context, term data needs a term context).
5. **Verify** by viewing the rendered result when possible; empty output almost always means: unregistered meta (`show_in_rest` missing, underscore key), missing context, an unactivated snippet, or a non-bindable attribute.

## Common mistakes

- Binding a non-bindable attribute (e.g. a Group's background image, a Cover's URL) — it is ignored.
- Meta key starting with `_` or registered without `show_in_rest` → empty value.
- Forgetting `single => true` → array instead of a scalar.
- Using `core/term-data` / `core/post-data` fields on a WordPress version older than 6.9.
- Binding `url` to a value that isn't an absolute/valid URL — sanitize in the source; an empty/invalid value drops the link.
- Putting PHP or JS in the block markup or in an HTML block — code always goes in a Code Snippet.
- Naming a pattern-override block without a unique `metadata.name`, or reusing the same name twice in one pattern.
- Expecting the fallback text to be stored as the "real" value — the binding resolves at render time.
