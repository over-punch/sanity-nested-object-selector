# sanity-nested-object-selector

[![npm version](https://img.shields.io/npm/v/@liiift-studio/sanity-nested-object-selector.svg)](https://www.npmjs.com/package/@liiift-studio/sanity-nested-object-selector)
[![license](https://img.shields.io/npm/l/@liiift-studio/sanity-nested-object-selector.svg)](https://www.npmjs.com/package/@liiift-studio/sanity-nested-object-selector)

Sanity Studio searchable checkbox selector for items nested within another document type. Builds GROQ queries dynamically and supports a title filter, single-click selection, and selection count feedback.

**Use it when** an editor needs to multi-select values that live *inside* the array of some other document — for example, picking specific `products` out of every `category` document, or `sections` out of every `licenseGroup` — without hand-writing a custom input per field. Point it at the source document type and the nested array via schema `options`; it fetches, flattens, and deduplicates the candidates for you, and stores the chosen values as a plain `string[]`.

## How it works

The component reads its configuration from the field's schema `options`, runs a generated GROQ query against your dataset, flattens the nested arrays from every matching document into `{ title, value }` items (deduplicated by `value`), and renders them as a checkbox list. The values you tick are written back to the field as a `string[]`.

![Data flow: source documents are projected and flattened into deduplicated title/value items, rendered as a checkbox selector, and stored as a string array of the selected values.](https://raw.githubusercontent.com/Liiift-Studio/sanity-nested-object-selector/main/assets/data-shape.svg?v=1)

> **Screenshot wanted.** This is a Studio UI component and cannot be captured headlessly. A maintainer screenshot/GIF of the selector in the Studio (search box + checked rows + "N selected" footer) should be added here. _Placeholder — see `assets/`._

## Install

```bash
npm install @liiift-studio/sanity-nested-object-selector
```

## Usage

Use `NestedObjectArraySelector` as a custom `input` component on an array field:

```typescript
import { NestedObjectArraySelector } from '@liiift-studio/sanity-nested-object-selector'

export const mySchema = defineType({
	name: 'collection',
	type: 'document',
	fields: [
		defineField({
			name: 'products',
			title: 'Products',
			type: 'array',
			of: [{ type: 'string' }],
			components: {
				input: NestedObjectArraySelector,
			},
			options: {
				sourceType: 'category',          // Document type that contains the nested array
				nestedField: 'products',         // Array field name within sourceType
				titleField: 'name',              // GROQ expression for the display label
				valueField: 'slug.current',      // GROQ expression for the stored value
				filter: 'status == "active"',    // Optional GROQ filter clause
				sortBy: 'name asc',              // Optional client-side sort (field + asc/desc)
				searchPlaceholder: 'Search products...',
				emptyMessage: 'No products found',
			},
		}),
	],
})
```

`titleField` and `valueField` are GROQ projection expressions evaluated **in the scope of each nested item**, so dotted paths like `slug.current` work. Each nested item must resolve to a **non-empty** `title` *and* `value` — items missing either are dropped. The stored field value is an array of the resolved `value` strings (e.g. `["oxford", "garamond"]`); when nothing is selected the field is `unset`.

### Options

| Option | Required | Description |
|---|---|---|
| `sourceType` | yes | Document `_type` to query. |
| `nestedField` | yes | Name of the array field within `sourceType` to flatten. |
| `titleField` | yes | GROQ expression (in nested-item scope) for the display label. |
| `valueField` | yes | GROQ expression (in nested-item scope) for the stored value. |
| `filter` | no | GROQ filter clause appended to the document query (e.g. `status == "active"`). |
| `sortBy` | no | Client-side sort as `"<field> <asc\|desc>"`. Defaults to `title asc`; non-`title` fields fall back to sorting by `title`. |
| `searchPlaceholder` | no | Placeholder for the search box. |
| `emptyMessage` | no | Message shown when the query returns no items. |

If any of the four required options is missing, the component renders a configuration-error card instead of the list.

### The `useNestedObjects` hook

The data layer is also exported on its own, in case you want to render the items with your own UI:

```javascript
import { useNestedObjects } from '@liiift-studio/sanity-nested-object-selector'

const { objects, loading, error } = useNestedObjects({
	sourceType: 'category',
	nestedField: 'products',
	titleField: 'name',
	valueField: 'slug.current',
	filter: 'status == "active"', // optional
	sortBy: 'name asc',           // optional
})
// objects is an array of { title: string, value: string }
```

It uses the Studio client (`useClient`, API version `2023-01-01`), so it must be called inside a Sanity Studio React tree.

> The package ships as compiled ESM/CJS without bundled `.d.ts` type declarations.

## Features

- Title-substring search box (appears once there are more than 5 items)
- Single-click checkbox selection
- Selected-count indicator
- Dynamic GROQ query generation across a source document type
- Automatic flattening and deduplication of nested items by value field

## GROQ note

The `sourceType`, `nestedField`, `titleField`, `valueField`, `filter`, and `sortBy` values are interpolated directly into the generated GROQ query. They are intended to come from **trusted schema configuration only** — do not wire them to end-user input.

## Requirements

This is a Sanity v3+ Studio plugin (built and tested against Sanity v5). It is a Studio input component, not a standalone library — it must run inside a Sanity Studio.

| Package | Version |
|---|---|
| `@sanity/ui` | `>=3` |
| `react` | `>=18` |
| `sanity` | `>=3` |

## License

MIT
