# sanity-nested-object-selector

[![npm version](https://img.shields.io/npm/v/@overpunch/sanity-nested-object-selector.svg)](https://www.npmjs.com/package/@overpunch/sanity-nested-object-selector)
[![license](https://img.shields.io/npm/l/@overpunch/sanity-nested-object-selector.svg)](https://www.npmjs.com/package/@overpunch/sanity-nested-object-selector)
[![sanity: v3 – v6](https://img.shields.io/badge/sanity-v3%20%E2%80%93%20v6-f03e2f.svg)](#requirements)

Sanity Studio searchable checkbox selector for items nested within another document type. Builds GROQ queries dynamically and supports a title filter, single-click selection, and selection count feedback.

**Use it when** an editor needs to multi-select values that live *inside* the array of some other document — for example, picking specific `products` out of every `category` document, or `sections` out of every `licenseGroup` — without hand-writing a custom input per field. Point it at the source document type and the nested array via schema `options`; it fetches, flattens, and deduplicates the candidates for you, and stores the chosen values as a plain `string[]`.

## How it works

The component reads its configuration from the field's schema `options`, runs a generated GROQ query against your dataset, flattens the nested arrays from every matching document into `{ title, value }` items (deduplicated by `value`), and renders them as a checkbox list. The values you tick are written back to the field as a `string[]`.

![Data flow: source documents are projected and flattened into deduplicated title/value items, rendered as a checkbox selector, and stored as a string array of the selected values.](https://raw.githubusercontent.com/over-punch/sanity-nested-object-selector/main/assets/data-shape.svg?v=1)

> **Screenshot wanted.** This is a Studio UI component and cannot be captured headlessly. A maintainer screenshot/GIF of the selector in the Studio (search box + checked rows + "N selected" footer) should be added here. _Placeholder — see `assets/`._

## Install

```bash
npm install @overpunch/sanity-nested-object-selector
```

## Usage

Use `NestedObjectArraySelector` as a custom `input` component on an array field:

```typescript
import { defineType, defineField } from 'sanity'
import { NestedObjectArraySelector } from '@overpunch/sanity-nested-object-selector'

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

### Data shape

The stored value is a plain `string[]` of the selected `valueField` results — no `_key`, no objects, no references:

```json
"products": ["oxford", "garamond"]
```

Because it is a plain string array, read it back with no dereferencing:

```groq
*[_type == "collection"]{ products }
```

If you need the full nested objects at query time, join back against the source type yourself:

```groq
*[_type == "collection"]{
  products,
  "resolved": *[_type == "category"].products[slug.current in ^.^.products]
}
```

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
import { useNestedObjects } from '@overpunch/sanity-nested-object-selector'

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

Supports **Sanity Studio v3, v4, v5 and v6** from a single build. It is a Studio input component, not a standalone library — it must run inside a Sanity Studio React tree (the `useNestedObjects` hook uses `useClient`, API version `2023-01-01`).

| Package | Supported range |
|---|---|
| `sanity` | `>=3 <7` (Studio v3 – v6) |
| `react` | `>=18` |
| `@sanity/ui` | `>=2 <5` |

This component uses no icons, so it declares **no `@sanity/icons` peer** — unlike its sibling packages.

### How one build spans four majors

The `@sanity/ui` peer range looks wrong at a glance, so here is the reasoning:

- **`@sanity/ui` v4 moved components to subpath entries.** `Tooltip`, `Menu`, `MenuButton`, `MenuItem`, `Code`, `Popover`, `Autocomplete`, `Toast` and `useToast` are no longer on the package root.
- **`@sanity/icons` v5 removed every named `*Icon` export** (relevant to the sibling packages in this suite, not to this one).
- **Both still *declare* the removed names in their `.d.ts`, typed `never`.** A named import therefore type-checks, compiles, and only then fails at runtime — the breakage is invisible to `tsc` and to a green build.
- **So this package imports no `@sanity/ui` symbol directly.** `Stack`, `Card`, `Text`, `Checkbox`, `Box`, `Spinner` and `Flex` all route through [`@overpunch/sanity-ui-compat`](https://www.npmjs.com/package/@overpunch/sanity-ui-compat) (a real runtime dependency, installed for you), which resolves the installed namespace at runtime and works against either layout.

**The `@sanity/ui` peer is `>=2 <5`, and that is correct for Sanity v6** — Studio v6 ships `@sanity/ui` **v4**, not v5. It is not a stale upper bound.

### Verification status

v3 – v6 support is established by the declared peer ranges, green builds, and the runtime-resolving compat layer. Beyond that, this component has been exercised in **three in-house Studios**. It has **not** been broadly tested in a running Sanity 6 Studio outside those. Please [open an issue](https://github.com/over-punch/sanity-nested-object-selector/issues) if you hit a version-specific problem.

### Packaging

- Ships **ESM** (`dist/index.mjs`) and **CJS** (`dist/index.js`).
- The build sets `dts: false`, so **no bundled `.d.ts` type declarations are shipped.** TypeScript consumers will need their own module declaration, or can import from the published `src/` (exposed via the `source` export condition).

## License

MIT
