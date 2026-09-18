# WordPress Abilities

Rules for registering AI-callable abilities (tools) with the
[WordPress Abilities API](https://github.com/WordPress/abilities-api).
These rules apply to any plugin exposing abilities — nothing here is
specific to a particular plugin.

## Where abilities live

Register abilities from a dedicated place (an "ability registrar" — a file
or function group), not inline from a controller or repository handler. The
registrar's job is only to wire up categories and abilities and call into
existing business logic (services/repositories/helper functions) from the
`execute_callback` — it should not contain business logic itself.

## 1. Register the category first

Every ability must belong to a category. Register categories on the
`wp_abilities_api_categories_init` hook and abilities on the
`wp_abilities_api_init` hook. Categories must exist before abilities
reference them, which is why the hooks are bound in that order:

```php
add_action('wp_abilities_api_categories_init', 'my_plugin_register_ability_categories');
add_action('wp_abilities_api_init', 'my_plugin_register_abilities');
```

```php
function my_plugin_register_ability_categories(): void
{
    if (! function_exists('wp_register_ability_category')) {
        return;
    }

    wp_register_ability_category(
        'my-plugin',
        [
            'label'       => __('My Plugin', 'my-plugin'),
            'description' => __('Abilities provided by My Plugin for AI agents.', 'my-plugin'),
        ]
    );
}
```

- Guard with `function_exists()` — the Abilities API may not be loaded
  (older WP core, or the feature not available yet).
- One category slug per plugin/sub-area is usually enough; every ability
  points at that same category slug. Only introduce a second category if
  the group of abilities is genuinely a different surface (see "Grouping"
  below) — splitting for readability is not a reason to add a new category.

## 2. Register abilities

```php
function my_plugin_register_abilities(): void
{
    if (! function_exists('wp_register_ability')) {
        return;
    }

    wp_register_ability(
        'my-plugin/get-widgets',
        [
            'label'               => __('Get Widgets', 'my-plugin'),
            'description'         => __('Returns the list of all widgets.', 'my-plugin'),
            'category'            => 'my-plugin',
            'execute_callback'    => function () {
                return my_plugin_get_widgets();
            },
            'permission_callback' => function () {
                return current_user_can('edit_posts');
            },
            'output_schema'       => [
                'type' => 'array',
            ],
            'meta'                => [
                'show_in_rest' => true,
                'annotations'  => ['readonly' => true],
            ],
        ]
    );
}
```

### Naming

Ability id is `<plugin-slug>/<kebab-case-verb-noun>`. Settle on a small,
consistent CRUD-style vocabulary and reuse it for every entity, e.g.:
`get-x`, `get-x-by-slug` (or `get-x-by-id`), `add-or-update-x`, `delete-x`.
Don't invent a different verb style per entity — a consistent vocabulary
lets the model guess unseen ability names from ones it has already seen.

### Required keys

| Key | Notes |
|---|---|
| `label` | Translatable, short. |
| `description` | Translatable. Describe what it returns/does, and call out anything a model would otherwise have to guess (e.g. "accepts an id or a slug", "not recoverable"). |
| `category` | The registrar's category slug. |
| `execute_callback` | Closure calling into a service/repository/helper function — never inline business logic here. |
| `permission_callback` | Always required, never omit. See "Permission level" below. |
| `output_schema` | Always include, even if loosely typed (`['type' => 'array']` / `['type' => 'object']` is fine when the shape is dynamic). |
| `meta.show_in_rest` | `true` for every ability that should be visible through REST. |
| `meta.annotations` | See "Annotations" below. |

### input_schema — read this before adding one

**If the callback takes no arguments, do not add `input_schema` at all —
not even an empty object.**

```php
'execute_callback'    => function () {
    return my_plugin_get_options();
},
'permission_callback' => function () {
    return current_user_can('edit_posts');
},
// No input_schema at all: this ability takes no arguments, and
// WP_Ability::validate_input() rejects the null input a provider
// sends for an argument-less tool call whenever a schema exists.
```

Why: `WP_Ability::validate_input()` rejects a `null` input whenever a
schema is present — but providers routinely call zero-argument tools with no
arguments at all (`null` input, not `{}`). If you attach a schema, even
`['type' => 'object', 'properties' => []]`, every argument-less call from a
real provider fails validation. Leave the key out entirely and let the
`execute_callback` either ignore its unused `$args` or take no parameter.

When the callback *does* take arguments, always define `input_schema` with
`type: object`, `properties`, and `required` where applicable:

```php
'input_schema' => [
    'type'       => 'object',
    'properties' => [
        'id'   => ['type' => 'integer', 'description' => __('Item ID.', 'my-plugin')],
        'slug' => ['type' => 'string', 'description' => __('Item slug, used when no id is given.', 'my-plugin')],
    ],
],
```

Give every property a `description` when its purpose or fallback behavior
isn't obvious from the name alone (e.g. "used when no id is given").

### id-or-slug lookup pattern

Get/delete abilities that target a single entity commonly accept both `id`
and `slug`, with `slug` used when `id` is absent:

```php
'execute_callback' => function ($args) {
    return my_plugin_get_item(
        (int) ($args['id'] ?? 0),
        (string) ($args['slug'] ?? '')
    );
},
```

Cast every arg pulled from `$args` — providers send loosely-typed JSON.

### Annotations

`meta.annotations` tells the model (and any UI) what kind of operation this
is:

- `readonly => true` for every read/list/get ability.
- `readonly => false` for every create/update ability.
- `readonly => false, destructive => true, idempotent => false` for every
  delete ability, since a second identical call is a no-op error, not a
  repeat of the same effect.

### Permission level

Choose the capability check based on what the ability actually touches, not
a single blanket default:

- Content the current user manages themselves (their own posts/entities of
  the plugin's custom post types) → typically `current_user_can('edit_posts')`.
- Site-wide or public-facing configuration, or anything that runs
  unattended and can call write abilities without a human approving each
  call (e.g. a public widget, a scheduled/cron task borrowing another
  user's capabilities) → escalate to `current_user_can('manage_options')`.
- An ability that is genuinely public — safe for anyone to call, including
  unauthenticated visitors (e.g. reading data already exposed on the public
  front end) → use `__return_true` instead of a capability check:

```php
wp_register_ability(
    'my-plugin/get-public-info',
    [
        'label'               => __('Get Public Info', 'my-plugin'),
        'description'         => __('Returns information already publicly visible on the site.', 'my-plugin'),
        'category'            => 'my-plugin',
        'execute_callback'    => function () {
            return my_plugin_get_public_info();
        },
        'permission_callback' => '__return_true',
        'output_schema'       => [
            'type' => 'object',
        ],
        'meta'                => [
            'show_in_rest' => true,
            'annotations'  => ['readonly' => true],
        ],
    ]
);
```

Only reach for `__return_true` when the data/action is meant to be public —
never as a shortcut to skip writing a real capability check. If in doubt,
require a capability.

When gating a whole sub-group at the stricter level, extract one shared
closure and reuse it:

```php
$can_manage = static function (): bool {
    return current_user_can('manage_options');
};
```

Document *why* a group needs the escalation (or the public grant) in a
comment above its registration block — don't leave the reader to guess why
this group differs from the rest of the file.

### Error results, not thrown exceptions

Abilities return data, they don't throw. A "not found" or "invalid input"
case returns an array with an `error` key that the model can read and relay,
and every failure branch reports through `success => false` plus `error`,
never a PHP exception:

```php
if (null === $item) {
    return ['error' => __('No matching item was found.', 'my-plugin')];
}
```

```php
if (! my_plugin_trash_widget($id)) {
    return ['success' => false, 'error' => __('The widget could not be trashed.', 'my-plugin')];
}
```

Reflect the `error` (and `success`) fields in `output_schema.properties` so
the shape is documented even on the failure path.

## Grouping abilities in a large registrar

When a registrar accumulates many abilities, split by entity into separate
functions called from the main registration function, one function per
entity/surface (e.g. `my_plugin_register_widget_abilities()`,
`my_plugin_register_scheduled_task_abilities()`). All groups still register
under the same category slug — splitting is purely for readability, not a
reason for a second category.

```php
function my_plugin_register_abilities(): void
{
    if (! function_exists('wp_register_ability')) {
        return;
    }

    wp_register_ability('my-plugin/get-items', [ /* ... */ ]);
    // ...

    my_plugin_register_widget_abilities();
    my_plugin_register_scheduled_task_abilities();
}
```

## Checklist for a new ability

1. Does the entity already have a service/repository/helper function to
   call? If not, add it there — not inline in the ability closure.
2. Pick the id: `<plugin-slug>/<verb>-<noun>` following the vocabulary
   already used elsewhere in the plugin (`get`, `get-x-by-slug`,
   `add-or-update`, `delete`).
3. Does the callback take arguments?
   - No → omit `input_schema` entirely.
   - Yes → define `input_schema` with typed `properties` and `required`.
4. Write `output_schema` matching the real return shape, including `error`
   (and `success` for mutating abilities).
5. Pick `permission_callback` based on what the ability touches — plugin
   content vs. site-wide/public-facing/unattended surfaces vs. genuinely
   public data (`__return_true`) — and document any escalation to
   `manage_options` or grant of `__return_true`.
6. Set `meta.annotations` (`readonly`, plus `destructive`/`idempotent` for
   deletes) and `meta.show_in_rest => true`.
7. Register under the registrar's existing category slug — never add a new
   category unless this is a genuinely new plugin surface.
