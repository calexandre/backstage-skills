# Pattern: Field schema with `makeFieldSchema`

The schema declares two things to the scaffolder framework:

- The **output** type — what gets stored in `formData` and emitted as the field's value.
- The **uiOptions** type (optional) — the shape of `ui:options` template authors can pass.

It's defined via `makeFieldSchema` from `@backstage/plugin-scaffolder-react` using Zod, and produces both a runtime schema and the `TProps` type alias for the component.

## Standalone usage

### Basic schema (output only)

```ts
// src/components/SlugPicker/schema.ts
import { makeFieldSchema } from '@backstage/plugin-scaffolder-react';

export const SlugPickerFieldSchema = makeFieldSchema({
  output: z => z.string(),
});

// Re-export typed props for the component to consume.
// `TProps` is the inferred FieldExtensionComponentProps with both generics resolved.
export type SlugPickerProps = typeof SlugPickerFieldSchema.TProps;

// Re-export the JSON-schema form for advanced template-editor use cases.
export const SlugPickerSchema = SlugPickerFieldSchema.schema;
```

In the component file, the type alias closes the loop:

```ts
import { SlugPickerProps } from './schema';

export const SlugPicker = (props: SlugPickerProps) => {
  /* ... */
};
```

### Schema with UI options

When templates can pass field-specific config via `ui:options`, declare the shape with `uiOptions`:

```ts
import { makeFieldSchema } from '@backstage/plugin-scaffolder-react';

export const SlugPickerFieldSchema = makeFieldSchema({
  output: z => z.string(),
  uiOptions: z =>
    z.object({
      allowUppercase: z
        .boolean()
        .optional()
        .describe('When true, the field preserves uppercase characters.'),
      maxLength: z.number().int().positive().optional(),
    }),
});

export type SlugPickerProps = typeof SlugPickerFieldSchema.TProps;
```

Templates then write:

```yaml
parameters:
  - properties:
      slug:
        type: string
        title: Slug
        ui:field: SlugPicker
        ui:options:
          allowUppercase: false
          maxLength: 32
```

Inside the component, `props.uiSchema['ui:options']` is typed as `{ allowUppercase?: boolean; maxLength?: number }`.

### Output that isn't a string

Fields can produce structured values too. Match the Zod type to whatever the field emits:

```ts
// A picker that emits { entityRef: string; namespace: string }
export const EntityChoiceFieldSchema = makeFieldSchema({
  output: z =>
    z.object({
      entityRef: z.string(),
      namespace: z.string(),
    }),
});
```

Templates consuming the field receive an object value:

```yaml
steps:
  - id: pick-entity
    name: Pick entity
    action: ...
    input:
      ref: ${{ parameters.entity.entityRef }}
      ns: ${{ parameters.entity.namespace }}
```

### Naming & file layout

- One schema file per field, alongside the component (`SlugPicker.tsx` + `schema.ts`).
- Validation, when used, lives in the same folder (`validation.ts`); see `references/field-validation.md`.
- Re-export `<Name>FieldSchema` (the schema object), `<Name>Schema` (the JSON-schema form), and `<Name>Props` (the typed props) so the component, validation, and registration files share types cleanly.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin-module`:

1. Create `plugins/<id>/src/components/<FieldName>/schema.ts` with at minimum an `output` Zod factory and a re-exported `<FieldName>Props` type.
2. If the user's intent describes per-template config ("templates should be able to pass max length", "the field has options for X"), add `uiOptions` with the inferred properties.
3. Add `@backstage/plugin-scaffolder-react` and `zod` to `dependencies`. Note: import Zod via the same v3 channel Backstage uses internally — `import { z } from 'zod/v3'` if your codebase pins Zod and otherwise plain `import { z } from 'zod'`. Verify against `../backstage/plugins/scaffolder-react/src/next/blueprints/FormFieldBlueprint.tsx`.

## Tests

Schema correctness is enforced by the type system (`TProps` is inferred), so dedicated unit tests are rarely needed. If the schema embeds non-trivial Zod constraints, write a small parse test:

```ts
import { SlugPickerFieldSchema } from './schema';

it('validates ui:options shape', () => {
  const ok = SlugPickerFieldSchema.uiOptionsType.safeParse({
    allowUppercase: true,
  });
  expect(ok.success).toBe(true);

  const bad = SlugPickerFieldSchema.uiOptionsType.safeParse({
    maxLength: -1,
  });
  expect(bad.success).toBe(false);
});
```

(Verify the exact accessor on `FieldSchema` — it may be `.uiOptionsType` or similar — against the upstream type definitions in `../backstage/plugins/scaffolder-react/src/types.ts`.)
