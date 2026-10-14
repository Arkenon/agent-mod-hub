**Title:** WordPress Interactivity API

**Slug:** wordpress-interactivity-api

**Description:** Use this skill whenever a page, pattern, template or block needs front-end interactivity built with the WordPress Interactivity API — `data-wp-*` directives, a store with state/actions, toggles, tabs, filters, counters, "load more", live fetches — or when existing directives "don't fire". Covers how to do it without files or a build step: directives go into block markup (a custom PHP block, a `render_block` filter on core blocks in a pattern/template, or a Custom HTML block), and the store, initial state and loader go into a single PHP Code Snippet that prints an inline script module.

**Version:** 1.0.0

===

## Writing code: use Code Snippets

You run inside WordPress, not in an IDE or terminal — you cannot edit theme or plugin files. Whenever this skill calls for code (PHP, JavaScript or CSS), deliver it as a Code Snippet using the `agent-mod-pro` abilities: `get-snippets` / `get-snippet` to look for an existing one first, `validate-snippet` to check the code, `add-snippet` to create it and `update-snippet` to change it (plus `set-snippet-status`, `remove-snippet` and the code-reading abilities when needed). Prefer updating an existing snippet over creating a duplicate. A new snippet is a reviewable draft: tell the user it takes effect only once it is activated. Explain what the code must do, then create the snippet — don't paste code into chat and ask the user to place it somewhere.

## What this is

The Interactivity API is WordPress's standard way to add front-end behaviour. It has two halves:

- **Directives** — `data-wp-*` attributes in the HTML that say *what* reacts: `data-wp-interactive="agentmod/faq"` marks a region and its store namespace, `data-wp-on--click="actions.toggle"` wires an event, `data-wp-bind--hidden="!context.isOpen"` binds an attribute to state.
- **A store** — JavaScript registered with `store( 'agentmod/faq', { state, actions, callbacks } )` from the `@wordpress/interactivity` script module. It holds the logic.

The server renders the initial HTML (it can evaluate directives in PHP so the first paint is already correct), then the client runtime **hydrates** the region on `DOMContentLoaded` and keeps the DOM in sync with state. Requires WordPress 6.5+; version-specific notes are marked inline.

Naming: use the store namespace `agentmod/<feature>` and prefix every PHP function with `agentmod_<feature>_` (e.g. `agentmod_faq_…`). One feature = one namespace = one snippet.

**Check core blocks first.** WordPress already ships interactive blocks — Accordion, Tabs, Details, Navigation (overlay menu), Image lightbox, Search, Query Loop (enhanced pagination). If the request is "an accordion" or "tabs", build it with those blocks (block markup skill) and don't write a store at all.

## Where each part lives in AgentMod

The upstream workflow (block.json, `view.js`, `viewScriptModule`, `@wordpress/scripts`, `create-block`) does not exist here. Map it like this:

| Part | Where it goes |
|------|---------------|
| Directives (`data-wp-*`) | Block markup output — see "Choose the integration path" |
| Store (state / actions / callbacks JS) | Inside the feature's **PHP** snippet, printed as an inline `<script type="module">` |
| Loading `@wordpress/interactivity` | Same PHP snippet — a "virtual" script module with no file (below) |
| Initial global state, derived state, translations, nonces | Same PHP snippet — `wp_interactivity_state()` |
| Local context for one element | `data-wp-context` in markup, or `wp_interactivity_data_wp_context()` in PHP |
| Styles (`.is-open`, transitions) | A separate `css` snippet (it reaches the front end and the editor canvas) |
| Region markup in a pattern / template / page | Block markup via the pattern, template and post abilities |

Never:

- Put store code in a **`js` snippet**. JS snippets are printed as one classic inline script on every page; `import { store } from '@wordpress/interactivity'` is a syntax error there, and a dynamic `import()` races hydration.
- Put a `<script>` with `import` into post content, a pattern or a Custom HTML block.
- Ask the user to create `block.json`, `view.js`, run a build, or edit theme files.

## The loader: a script module without a file

A script module normally needs a file URL. Instead, register a module with an **empty `src`** that only depends on `@wordpress/interactivity`. WordPress never prints the empty module itself, but its dependency goes into the import map (plus a modulepreload and the server state JSON). Then print the store as an inline module in the footer, after the import map:

1. On `init`: `wp_register_script_module( 'agentmod/<feature>', '', array( '@wordpress/interactivity' ) );`
2. Only when the feature is actually rendered on the page (inside the render callback / `render_block` filter): `wp_enqueue_script_module( 'agentmod/<feature>' )` and `add_action( 'wp_footer', 'agentmod_<feature>_print_store', 20 )`. Both calls are idempotent, so a feature rendered many times loads once.
3. In the footer callback: `wp_print_inline_script_tag( $js, array( 'type' => 'module', 'id' => 'agentmod-<feature>-view-js' ) )`. Priority 20 keeps it after the import map (classic themes print the map at `wp_footer` 10; block themes in `wp_head`). Inline modules run before `DOMContentLoaded`, so the store exists before hydration.

Write the JS in a **nowdoc** (`<<<'JS'` … `JS;`) so PHP does not interpolate `$` or need escaping. The closing `JS;` must be on its own line. Static `import` statements only, at the top of the module.

## Choose the integration path

Static core blocks (group, paragraph, heading, button, …) save their HTML into the post. **Never add `data-wp-*` attributes to that saved HTML** — the editor will flag the block as invalid ("This block contains unexpected or invalid content") and may strip them. Directives must be produced at render time or live in raw HTML. Three ways:

### A. Custom PHP block — reusable, data-driven UI

Best when the UI needs server data (posts, meta, options, REST), settings, or is reused across many pages. Register a dynamic block in the snippet (see the PHP-only block skill for `autoRegister`, attributes and limits) with `'supports' => array( 'interactivity' => true, … )`. WordPress then evaluates the directives in the `render_callback` output on the server automatically — don't call `wp_interactivity_process_directives()` yourself.

- In `render_callback`: set state with `wp_interactivity_state()`, build markup with directives (escape every dynamic value), use `get_block_wrapper_attributes()` on the wrapper, put local context with `wp_interactivity_data_wp_context()`, and call the enqueue helper.
- Place it in patterns/templates/pages as a self-closing dynamic block: `<!-- wp:agentmod/faq {"title":"FAQ"} /-->`.
- Snippet scope **global** (the editor preview renders it over REST). The editor preview shows the server-rendered first state only; clicks don't run there.

### B. Enhance core blocks in a pattern or template

Best when the design is built from normal blocks (so it stays editable) and only needs behaviour. The markup carries nothing but a **marker class** in each block's `className` (and, for static blocks, in the saved HTML `class` too — follow the block markup skill exactly). A `render_block` filter in the snippet finds the marker on the front end, injects the directives with `WP_HTML_Tag_Processor`, sets state, and runs the server pass once on that root:

```html
<!-- wp:group {"className":"agentmod-reveal","layout":{"type":"constrained"}} -->
<div class="wp-block-group agentmod-reveal">
<!-- wp:buttons -->
<div class="wp-block-buttons"><!-- wp:button {"tagName":"button","className":"agentmod-reveal__toggle"} -->
<div class="wp-block-button agentmod-reveal__toggle"><button type="button" class="wp-block-button__link wp-element-button">Show details</button></div>
<!-- /wp:button --></div>
<!-- /wp:buttons -->

<!-- wp:group {"className":"agentmod-reveal__panel","layout":{"type":"constrained"}} -->
<div class="wp-block-group agentmod-reveal__panel"><!-- wp:paragraph -->
<p>Hidden content the editor can still edit.</p>
<!-- /wp:paragraph --></div>
<!-- /wp:group --></div>
<!-- /wp:group -->
```

Rules for path B:

- Hook the specific block (`render_block_core/group`) and check the marker in `$block['attrs']['className']` before doing anything; return `$block_content` untouched otherwise.
- Do all injection in the **root** marker's filter (its `$block_content` already contains the rendered inner blocks), walking with `next_tag( array( 'class_name' => '…' ) )`. Add `data-wp-interactive` and `data-wp-context` (value `wp_json_encode( … )`) on the root.
- Use a `<button>` for toggles (`core/button` with `"tagName":"button"`), and bind `aria-expanded` / `aria-controls` / `hidden` — not a heading or a link without `href`.
- Call `wp_interactivity_process_directives()` on the root HTML exactly once. **Don't** use path B when the marked root sits inside, or wraps, a core interactive block (Query Loop, Terms Query, Accordion, Tabs, Navigation, Search, File, Playlist, Image with lightbox): those are already processed as interactive roots and would be processed twice. In that case skip `process_directives()` and set the initial attributes yourself (e.g. add `hidden` with `set_attribute`) — or use path A.
- Snippet scope **front-end** is enough (editor never needs it).

### C. Custom HTML block — one-off, designer-controlled UI

Best for a single hand-built widget. Write the directives directly into the raw HTML of a `<!-- wp:html -->` block; on WordPress 7.1+ it can contain editable paragraph/heading/image slots (see the editable HTML block skill). Raw HTML is not validated against a `save()`, so the attributes survive.

```html
<!-- wp:html -->
<div class="agentmod-counter" data-wp-interactive="agentmod/counter" data-wp-context='{"count":0}'>
  <button type="button" data-wp-on--click="actions.increment">+1</button>
  <output data-wp-text="context.count">0</output>
</div>
<!-- /wp:html -->
```

- Write `data-wp-context` as single-quoted attribute containing JSON.
- There is no server pass: **hand-write the initial HTML to match the initial state** (`0` inside the output, `hidden` on closed panels, `aria-expanded="false"`), otherwise it flickers on hydration.
- Enqueue the loader only on pages that use it: hook `render_block_core/html` and check `str_contains( $block_content, 'data-wp-interactive="agentmod/counter"' )`.
- Don't put `<script>` in the HTML block for this — the store lives in the snippet.

## Example snippet (path B: "reveal" toggle)

Function-based, shown for reference — put it in a snippet, never in a file. No opening `<?php` tag, every function guarded by `function_exists()`, every hook callback a named function (never a class or closure as a callback).

```php
if ( ! function_exists( 'agentmod_reveal_register' ) ) {
	function agentmod_reveal_register() {
		// Virtual module: no file, it only pulls @wordpress/interactivity into the import map.
		wp_register_script_module( 'agentmod/reveal', '', array( '@wordpress/interactivity' ) );
	}
	add_action( 'init', 'agentmod_reveal_register' );
}

if ( ! function_exists( 'agentmod_reveal_enqueue' ) ) {
	function agentmod_reveal_enqueue() {
		wp_enqueue_script_module( 'agentmod/reveal' );
		add_action( 'wp_footer', 'agentmod_reveal_print_store', 20 );
	}
}

if ( ! function_exists( 'agentmod_reveal_print_store' ) ) {
	function agentmod_reveal_print_store() {
		$js = <<<'JS'
import { store, getContext } from '@wordpress/interactivity';

const { state } = store( 'agentmod/reveal', {
	state: {
		get toggleLabel() {
			return getContext().isOpen ? state.labels.hide : state.labels.show;
		},
	},
	actions: {
		toggle() {
			const context = getContext();
			context.isOpen = ! context.isOpen;
		},
	},
} );
JS;
		wp_print_inline_script_tag( $js, array( 'type' => 'module', 'id' => 'agentmod-reveal-view-js' ) );
	}
}

if ( ! function_exists( 'agentmod_reveal_render' ) ) {
	function agentmod_reveal_render( $block_content, $block ) {
		$class = isset( $block['attrs']['className'] ) ? (string) $block['attrs']['className'] : '';
		if ( ! preg_match( '/(^|\s)agentmod-reveal(\s|$)/', $class ) ) {
			return $block_content;
		}

		wp_interactivity_state(
			'agentmod/reveal',
			array(
				'labels'      => array(
					'show' => __( 'Show details', 'agentmod' ),
					'hide' => __( 'Hide details', 'agentmod' ),
				),
				// Derived state mirrored in PHP so the first paint is right; the JS getter has the same name.
				'toggleLabel' => function () {
					$state   = wp_interactivity_state( 'agentmod/reveal' );
					$context = wp_interactivity_get_context( 'agentmod/reveal' );
					return ! empty( $context['isOpen'] ) ? $state['labels']['hide'] : $state['labels']['show'];
				},
			)
		);

		static $instance = 0;
		$panel_id = 'agentmod-reveal-panel-' . ( ++$instance );

		$p = new WP_HTML_Tag_Processor( $block_content );
		if ( ! $p->next_tag( array( 'class_name' => 'agentmod-reveal' ) ) ) {
			return $block_content;
		}
		$p->set_attribute( 'data-wp-interactive', 'agentmod/reveal' );
		$p->set_attribute( 'data-wp-context', wp_json_encode( array( 'isOpen' => false ) ) );

		if ( $p->next_tag( array( 'class_name' => 'agentmod-reveal__toggle' ) ) && $p->next_tag( 'button' ) ) {
			$p->set_attribute( 'aria-controls', $panel_id );
			$p->set_attribute( 'data-wp-on--click', 'actions.toggle' );
			$p->set_attribute( 'data-wp-bind--aria-expanded', 'context.isOpen' );
			$p->set_attribute( 'data-wp-text', 'state.toggleLabel' );
		}
		if ( $p->next_tag( array( 'class_name' => 'agentmod-reveal__panel' ) ) ) {
			$p->set_attribute( 'id', $panel_id );
			$p->set_attribute( 'data-wp-bind--hidden', '!context.isOpen' );
		}

		agentmod_reveal_enqueue();

		// Server pass: renders aria-expanded="false", the label and the hidden attribute before JS loads.
		return wp_interactivity_process_directives( $p->get_updated_html() );
	}
	add_filter( 'render_block_core/group', 'agentmod_reveal_render', 10, 2 );
}
```

For path A the same register / enqueue / print-store functions are reused; `render_callback` builds the markup, sets state and calls `agentmod_<feature>_enqueue()`. For path C the `render_block_core/html` filter only calls the enqueue helper (and may set state) and returns the content unchanged.

## Server-side rendering and state

- **Global state** (`wp_interactivity_state( 'ns', array( … ) )`) — shared by every region of that namespace; serialized to the client automatically. Read it in JS through the `state` returned by `store()`.
- **Local context** (`data-wp-context`) — per element and inherited by its children; read with `getContext()`. Use it for per-instance values (`isOpen`, an item id). In PHP, `wp_interactivity_data_wp_context( $array )` returns the escaped attribute.
- **Derived state** — values computed from other state (`hasItems`, a label). If it affects the first paint (`hidden`, text, classes), define it in PHP too, otherwise the server renders it as missing and the page shifts on hydration. Static values: compute and pass them. Context-dependent values: a closure that reads `wp_interactivity_state()` / `wp_interactivity_get_context()`; it is evaluated during directive processing. The closure's code is not sent to the browser (WordPress only flags it as a derived value), so **the JS store must define a getter with the same name and the same logic** — otherwise it is empty on the client after the first interaction.
- **Server values for the client** — translations (`__()`), `rest_url( 'agentmod/v1/' )`, `admin_url( 'admin-ajax.php' )`, `wp_create_nonce( 'wp_rest' )` go into state. Values that never change and need no reactivity can go into `wp_interactivity_config( 'ns', array( … ) )` and be read with `getConfig()`.
- State must be set **before** directives are processed (in the render callback / filter, before `process_directives()` or before returning from a path-A render callback).
- `wp_interactivity_process_directives()` runs once, at the outermost level of a region — never on a fragment that is part of an already processed root.
- Anything a fetch action calls (REST route, `admin-ajax` handler) is PHP too: add it to the same feature snippet, with a capability/nonce check and sanitized input.

## Directives quick reference

| Directive | Purpose |
|-----------|---------|
| `data-wp-interactive="ns"` | Declares an interactive region and its store namespace |
| `data-wp-context='{"key":value}'` | Local context for the element and its descendants |
| `data-wp-bind--attr="state.x"` | Sets/removes an attribute (`hidden`, `disabled`, `aria-expanded`, `href`, `src`…) |
| `data-wp-class--name="context.isOpen"` | Toggles a class |
| `data-wp-style--prop="state.color"` | Sets an inline style property |
| `data-wp-text="state.label"` | Sets the element's text (escaped) |
| `data-wp-on--event="actions.fn"` | Event handler (`click`, `submit`, `input`, `keydown`…) |
| `data-wp-on-async--event` | Async handler, preferred when the handler needs no synchronous event access |
| `data-wp-on-window--event` / `data-wp-on-document--event` | Global listeners (`resize`, `keydown`, `scroll`); `-async` variants exist |
| `data-wp-init="callbacks.fn"` | Runs once when the element is hydrated |
| `data-wp-watch="callbacks.fn"` | Runs on hydration and whenever the state it reads changes |
| `data-wp-each="state.items"` on a `<template>` | Renders a list; item is `context.item` (rename with `data-wp-each--name`); add `data-wp-each-key="context.item.id"` |
| `data-wp-key` | Stable key for elements in a list |

- References are `state.x`, `context.x`, `actions.x`, `callbacks.x`; prefix with `!` to negate. Another namespace: `other/ns::state.x`.
- Unique directive IDs (6.9+): `data-wp-on--click---agentmod-a="…"` and `data-wp-on--click---agentmod-b="…"` can share one element; the suffix after `---` must be unique per element.
- `data-wp-ignore` is deprecated (6.9) — never use it; split the region instead.
- Store API: `store`, `getContext`, `getElement` (current element `ref` / attributes), `getConfig`, `getServerState` / `getServerContext`. Async actions are generator functions — `*fetchItems() { const res = yield fetch( state.restUrl ); … }` — not `async`/`await`, so context stays correct.
- Handlers run asynchronously by default (6.8+): an action that needs `event.preventDefault()`, `stopPropagation()` or `event.currentTarget` must be wrapped with `withSyncEvent( ( event ) => { … } )` (import it from `@wordpress/interactivity`).
- Client-side navigation (6.9): `getServerState()` / `getServerContext()` reset on each page transition — don't assume old values persist.

## Workflow

1. **Clarify the behaviour** and check if a core interactive block already does it.
2. **Look for existing work**: `get-snippets` for an `agentmod_<feature>` snippet, and the target page/pattern/template markup. Update instead of duplicating.
3. **Pick the path**: data-driven or reused → A; behaviour on a normal block design → B; one-off hand-built widget → C.
4. **Create the feature snippet** (`validate-snippet`, then `add-snippet`): loader + store + state + render hook. Choose scope (A: global, B/C: front-end). Tell the user it is a draft that works only after activation.
5. **Write the markup** with the pattern / template / post abilities, following the block markup skill (and patterns/templates skills when relevant). Path B: only marker classes; path A: the block comment; path C: the HTML block.
6. **Styles** in a separate `css` snippet if needed (scope them under the marker / root class).
7. **Hand off**: say clearly that interactivity runs only on the front end (and preview) — never in the editor canvas — and how to test it.

## Debugging — "nothing happens"

1. Is the snippet **active**? A draft does nothing. Is the render hook/condition actually hit (right block name, marker class spelled the same in `className`)?
2. Does the rendered page HTML contain `data-wp-interactive="agentmod/<feature>"` on the root?
3. Does the page source contain the import map with `@wordpress/interactivity` and the inline `<script type="module" id="agentmod-<feature>-view-js">` **after** it? Missing map entry → `wp_enqueue_script_module()` was not called during render. Script before the map → footer priority too low.
4. Does the namespace in `store( '…' )` match `data-wp-interactive` exactly?
5. Browser console: any error before interaction (syntax error in the nowdoc JS, `Failed to resolve module specifier`)?
6. Narrow it down: remove directives one by one to find the broken reference (typo in `actions.x`, a missing getter).

Other symptoms:

- **Flicker / layout shift on load** — derived state or the initial attribute missing on the server: mirror it in PHP (path A/B) or hand-write it (path C).
- **Initial values wrong but clicks work** — state not set before processing, or path C without hand-written initial HTML, or path A block without `supports.interactivity`.
- **"Block contains unexpected or invalid content"** in the editor — directives were written into a static block's saved HTML; move them to a render filter (path B).
- **Directives processed twice / odd output** — a path-B root inside or around a core interactive block; drop `process_directives()` there or switch to path A.
- **Two plugins handling the same event** — use unique directive IDs (`---suffix`).

## Common mistakes

- Store code in a `js` snippet, a `<script>` in content, or a request to create `view.js` / `block.json`.
- `data-wp-*` written into the saved HTML of core blocks instead of a render filter.
- Forgetting the enqueue call during render → no import map entry, store never loads.
- Namespace mismatch between markup and `store()`.
- Derived state defined only in JS (layout shift) or only in PHP (missing on the client).
- Calling `wp_interactivity_process_directives()` more than once on the same region, or on a region nested in a core interactive block.
- `async`/`await` actions instead of generator actions; `preventDefault()` without `withSyncEvent`.
- Using `data-wp-ignore`.
- Not escaping dynamic values in PHP-built markup (`esc_html`, `esc_attr`, `esc_url`); `wp_json_encode()` / `wp_interactivity_data_wp_context()` for context.
- Building a custom accordion/tabs when the core Accordion/Tabs/Details block covers it.
