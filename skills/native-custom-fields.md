**Title:** Native Custom Fields Data

**Slug:** native-custom-fields

**Description:** How to read and write the values of fields created with the Native Custom Fields (NCF) plugin from code — meta keys, the value shape of every field type, and a ready-made pattern for create / update / delete abilities of an NCF post type that reads the field list at runtime instead of hard-coding it.

**Version:** 1.0.0

===

## Writing code: use Code Snippets

You run inside WordPress, not in an IDE or terminal — you cannot edit theme or plugin files. Whenever this skill calls for code (PHP, JavaScript or CSS), deliver it as a Code Snippet using the `agent-mod-pro` abilities: `get-snippets` / `get-snippet` to look for an existing one first, `validate-snippet` to check the code, `add-snippet` to create it and `update-snippet` to change it (plus `set-snippet-status`, `remove-snippet` and the code-reading abilities when needed). Prefer updating an existing snippet over creating a duplicate. A new snippet is a reviewable draft: tell the user it takes effect only once it is activated. Explain what the code must do, then create the snippet — don't paste code into chat and ask the user to place it somewhere.

## When to use this skill

Use it whenever code has to read or write data of a post type, taxonomy, user or options page whose fields were built with Native Custom Fields — for example "write create / update / delete abilities for the Service Provider post type". Combine it with the `wordpress-abilities` skill for the ability registration rules.

## Do not research NCF again — everything you need is here

Do not fetch the NCF documentation or read NCF plugin source for these tasks: the facts below are verified against the plugin source. Your tool-call budget is limited and old tool results are shortened as the conversation grows, so spend calls on the site's own data, not on documentation.

## Where things live

| What | Where |
|---|---|
| Field configuration (the builder schema) | `wp_options`: `native_custom_fields_post_meta_fields_config` (post types), keyed by post type slug → `sections[]` → `fields[]` |
| Post field values | `wp_postmeta`, meta key = the field's `name`, one row per field (`single => true`) |
| Term / user field values | `wp_termmeta` / `wp_usermeta`, same rules |

Every field is registered with `register_post_meta( $post_type, $name, [ 'type' => …, 'single' => true, 'show_in_rest' => … ] )`, so `get_registered_meta_keys( 'post', $post_type )` returns every meta key with its `type` and, for arrays/objects, its REST `schema`. The data is plain WordPress meta: read it with `get_post_meta( $id, $key, true )`, write it with `update_post_meta()` or `meta_input`.

A field in the stored configuration is a flat array: `name`, `fieldType`, `fieldLabel`, plus type options such as `options` and `multiple`, and `fields` (sub-fields) for `group` and `repeater`.

## The golden rule: read the field list at runtime

Do **not** hard-code the field list, types or options into the code you write. Read them when the code runs (see the template below). This means:

1. The code keeps working when someone adds or renames a field in the builder.
2. You do not need to keep the full field configuration in your context. Call `native-custom-fields/get-post-meta-fields` **once**, only to see the field names, types and labels so you can describe the abilities, and right away write a short table (name · fieldType · stored type · options) into your reply — that table is what you work from.

## Stored value shape per field type

| fieldType | Stored type | Value shape |
|---|---|---|
| `text`, `textarea`, `input`, `select` (single), `radio`, `combobox`, `toggle_group`, `date_picker`, `date_time_picker`, `time_picker`, `color_picker`, `color_palette`, `font_size`, `unit`, … | string | `"value"` |
| `number`, `range` | number | `42`, `3.5` — never a numeric string |
| `toggle`, `checkbox` | boolean | `true` / `false` — never `"1"` / `"yes"` |
| `select` with `multiple`, `token_field` | array | `["a", "b"]` |
| `group` | object | `{"sub_name": value, …}` — one object keyed by sub-field name |
| `repeater` | array | `[{"sub_name": value, …}, …]` — a list of row objects |
| `file_upload`, `media_library` | array | `[{"file_name": "photo.jpg", "file_to_upload": "https://…/photo.jpg", "file_size": "120.4 KB"}]` — always a list, even for one file; `file_to_upload` holds the URL |
| `notice`, `heading`, `text_highlight`, `section` | — | display only, no stored value: skip them |

- Sub-field values inside `group` / `repeater` follow the same table, recursively. The builder abilities create at most two nesting levels.
- `select` / `radio` / `combobox` / `toggle_group` options are configured as `"Label:value, Label 2:value2"`. The **value** part is stored, not the label.
- The NCF documentation describes media values as `{id, url, type, alt}`. The plugin actually stores the `file_name / file_to_upload / file_size` list above. If an existing post has a value, check it with `get_post_meta()` before relying on either shape.
- Pass arrays and objects to `update_post_meta()` as PHP arrays. WordPress serializes them. Never `json_encode()` them yourself.
- `update_post_meta()` and `meta_input` unslash their value, so pass `wp_slash( $value )` or backslashes in user text are lost.
- NCF's own `save_post` handler only runs for its meta box form (it checks a nonce), so programmatic writes are not overwritten.

## Sanitizing and validating

NCF ships a public sanitizer that knows every field type and recurses into groups and repeaters:

```php
\NativeCustomFields\Common\Helper::sanitizeFieldValue( $value, $field['fieldType'], $field['fields'] ?? [], $field['name'] );
```

Use it when the class exists, then validate the result against the registered schema with `rest_validate_value_from_schema()`. Reject unknown keys instead of saving them.

## Template: create / update / delete abilities for an NCF post type

Replace `my_site` with a short prefix for the site, `service_provider` with the post type, and the category with the one the site's other abilities use (register it only if none exists — see `wordpress-abilities`). The input schema of the `meta` argument is built from the live field configuration, so the model calling the ability sees every field with its type and options.

```php
if ( ! function_exists( 'my_site_ncf_fields' ) ) {
	/**
	 * Returns the value-bearing NCF fields of a post type, keyed by meta key.
	 */
	function my_site_ncf_fields( string $post_type ): array {
		$config = get_option( 'native_custom_fields_post_meta_fields_config', [] );
		$config = apply_filters( 'native_custom_fields_post_meta_fields', is_array( $config ) ? $config : [] );
		$fields = [];

		foreach ( $config[ $post_type ]['sections'] ?? [] as $section ) {
			foreach ( $section['fields'] ?? [] as $field ) {
				$type = $field['fieldType'] ?? '';

				if ( empty( $field['name'] ) || in_array( $type, [ 'notice', 'heading', 'text_highlight', 'section', 'meta_box' ], true ) ) {
					continue;
				}

				$fields[ $field['name'] ] = $field;
			}
		}

		return $fields;
	}
}

if ( ! function_exists( 'my_site_ncf_meta_schema' ) ) {
	/**
	 * Builds the JSON schema of the "meta" argument from the registered meta keys.
	 */
	function my_site_ncf_meta_schema( string $post_type ): array {
		$registered = get_registered_meta_keys( 'post', $post_type );
		$properties = [];

		foreach ( my_site_ncf_fields( $post_type ) as $name => $field ) {
			$type = $field['fieldType'] ?? 'text';
			$args = $registered[ $name ] ?? [];

			// Fallback when the meta is not registered yet: derive the type from fieldType.
			$fallback = 'string';
			if ( 'group' === $type ) {
				$fallback = 'object';
			} elseif ( in_array( $type, [ 'repeater', 'token_field', 'file_upload', 'media_library' ], true ) || ( 'select' === $type && ! empty( $field['multiple'] ) ) ) {
				$fallback = 'array';
			} elseif ( in_array( $type, [ 'toggle', 'checkbox' ], true ) ) {
				$fallback = 'boolean';
			} elseif ( in_array( $type, [ 'number', 'range' ], true ) ) {
				$fallback = 'number';
			}

			$schema = is_array( $args['show_in_rest'] ?? null ) && isset( $args['show_in_rest']['schema'] )
				? $args['show_in_rest']['schema']
				: [ 'type' => $args['type'] ?? $fallback ];

			$description = ( $field['fieldLabel'] ?? $name ) . ' (' . $type . ')';

			if ( ! empty( $field['options'] ) && is_string( $field['options'] ) ) {
				$description .= '. Allowed values (Label:value): ' . $field['options'];
			}

			$schema['description'] = $description;
			$properties[ $name ]   = $schema;
		}

		return [
			'type'                 => 'object',
			'properties'           => $properties,
			'additionalProperties' => false,
		];
	}
}

if ( ! function_exists( 'my_site_ncf_prepare_meta' ) ) {
	/**
	 * Sanitizes and validates meta input. Returns the clean values or a WP_Error.
	 */
	function my_site_ncf_prepare_meta( string $post_type, array $meta ) {
		$fields  = my_site_ncf_fields( $post_type );
		$schema  = my_site_ncf_meta_schema( $post_type );
		$unknown = array_diff( array_keys( $meta ), array_keys( $fields ) );

		if ( ! empty( $unknown ) ) {
			return new WP_Error( 'unknown_fields', 'Unknown fields: ' . implode( ', ', $unknown ) );
		}

		$clean = [];

		foreach ( $meta as $name => $value ) {
			$field = $fields[ $name ];

			if ( class_exists( '\NativeCustomFields\Common\Helper' ) ) {
				$value = \NativeCustomFields\Common\Helper::sanitizeFieldValue( $value, $field['fieldType'] ?? 'text', $field['fields'] ?? [], $name );
			}

			$valid = rest_validate_value_from_schema( $value, $schema['properties'][ $name ], $name );

			if ( is_wp_error( $valid ) ) {
				return $valid;
			}

			$clean[ $name ] = wp_slash( rest_sanitize_value_from_schema( $value, $schema['properties'][ $name ], $name ) );
		}

		return $clean;
	}
}

if ( ! function_exists( 'my_site_register_service_provider_abilities' ) ) {
	function my_site_register_service_provider_abilities(): void {
		if ( ! function_exists( 'wp_register_ability' ) ) {
			return;
		}

		$post_type = 'service_provider';
		$meta      = my_site_ncf_meta_schema( $post_type );
		$result    = [
			'type'       => 'object',
			'properties' => [
				'success' => [ 'type' => 'boolean' ],
				'post_id' => [ 'type' => 'integer' ],
				'meta'    => [ 'type' => 'object' ],
				'error'   => [ 'type' => 'string' ],
			],
		];

		wp_register_ability(
			'my-site/create-service-provider',
			[
				'label'               => __( 'Create Service Provider', 'my-site' ),
				'description'         => __( 'Creates a Service Provider post with its custom field values. Fields not given are left empty.', 'my-site' ),
				'category'            => 'my-site',
				'input_schema'        => [
					'type'       => 'object',
					'properties' => [
						'title'   => [ 'type' => 'string' ],
						'content' => [ 'type' => 'string' ],
						'status'  => [ 'type' => 'string', 'enum' => [ 'draft', 'publish', 'pending', 'private' ], 'default' => 'draft' ],
						'meta'    => $meta,
					],
					'required'   => [ 'title' ],
				],
				'output_schema'       => $result,
				'execute_callback'    => function ( $input ) use ( $post_type ) {
					$meta = my_site_ncf_prepare_meta( $post_type, (array) ( $input['meta'] ?? [] ) );

					if ( is_wp_error( $meta ) ) {
						return [ 'success' => false, 'error' => $meta->get_error_message() ];
					}

					$post_id = wp_insert_post(
						[
							'post_type'    => $post_type,
							'post_title'   => sanitize_text_field( (string) $input['title'] ),
							'post_content' => wp_kses_post( (string) ( $input['content'] ?? '' ) ),
							'post_status'  => sanitize_key( (string) ( $input['status'] ?? 'draft' ) ),
							'meta_input'   => $meta,
						],
						true
					);

					if ( is_wp_error( $post_id ) ) {
						return [ 'success' => false, 'error' => $post_id->get_error_message() ];
					}

					return [ 'success' => true, 'post_id' => $post_id, 'meta' => array_map( 'wp_unslash', $meta ) ];
				},
				'permission_callback' => function () use ( $post_type ) {
					$type = get_post_type_object( $post_type );

					return $type && current_user_can( $type->cap->create_posts );
				},
				'meta'                => [
					'show_in_rest' => true,
					'annotations'  => [ 'readonly' => false ],
				],
			]
		);

		// update-service-provider: same shape with a required integer "id"; check
		// get_post_type( $id ) === $post_type and current_user_can( 'edit_post', $id ),
		// call wp_update_post() for title / content / status when given and
		// update_post_meta() for each prepared meta value. Only the fields passed
		// are changed.

		// delete-service-provider: required integer "id" and optional boolean
		// "force" (default false = move to trash); check the post type and
		// current_user_can( 'delete_post', $id ), then wp_trash_post() or
		// wp_delete_post( $id, true ). Annotations: readonly false,
		// destructive true, idempotent false.
	}

	add_action( 'wp_abilities_api_init', 'my_site_register_service_provider_abilities' );
}
```

Write the update and delete abilities out in full in the snippet. The comments above only list what they must do.

## Checklist

1. Read the configuration once (`native-custom-fields/get-post-meta-fields`) and write the field table into your reply.
2. Do not hard-code fields: use `my_site_ncf_fields()` / `my_site_ncf_meta_schema()`.
3. Check each value type against the table: booleans are booleans, numbers are numbers, media is a list of file objects, a repeater is a list of row objects.
4. Sanitize with NCF's `Helper::sanitizeFieldValue()`, validate against the registered schema, reject unknown keys, and `wp_slash()` before saving.
5. Follow `wordpress-abilities` for annotations, permissions, `output_schema` and `error` / `success` results.
6. Validate the snippet with `validate-snippet`, save it inactive, and tell the user to activate it.
