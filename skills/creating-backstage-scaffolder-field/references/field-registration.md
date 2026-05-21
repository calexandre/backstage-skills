# Pattern: Field registration

Once you have the component, schema, and (optional) validation, two final pieces wire the field into the scaffolder:

1. **`createFormField`** — bundles the component + schema + validation into a single `FormField` object.
2. **`FormFieldBlueprint.make`** — registers the bundle as a plugin extension. Templates discover the field by name from the resulting extension.

Both come from `@backstage/plugin-scaffolder-react/alpha` (currently `@alpha`).

## Standalone usage

### `src/components/SlugPicker/index.ts`

```ts
import { createFormField } from '@backstage/plugin-scaffolder-react/alpha';
import { SlugPicker as Component } from './SlugPicker';
import { SlugPickerFieldSchema } from './schema';
import { slugPickerValidation } from './validation';

export const SlugPicker = createFormField({
  name: 'SlugPicker',
  component: Component,
  schema: SlugPickerFieldSchema,
  validation: slugPickerValidation,
});
```

Notes:

- `name` MUST match what templates write in `ui:field`. Use PascalCase (matches the upstream-built-in fields like `EntityPicker`, `RepoUrlPicker`).
- `validation` is optional — omit if the schema alone is sufficient.

### `src/extensions.tsx`

Wrap the field in a `FormFieldBlueprint` extension. The blueprint's `params.field` is a function that returns a Promise resolving to the `FormField` — this enables code-splitting so unused field plugins don't bloat the initial bundle:

```tsx
import { FormFieldBlueprint } from '@backstage/plugin-scaffolder-react/alpha';

export const slugPickerFormField = FormFieldBlueprint.make({
  name: 'slug-picker',
  params: {
    field: () => import('./components/SlugPicker').then(m => m.SlugPicker),
  },
});
```

The blueprint's own `name` is the extension id (kebab-case, used internally by NFS). The user-facing field name is set in `createFormField`.

### `src/plugin.tsx`

Add the extension to the plugin's `extensions` array:

```tsx
import { createFrontendPlugin } from '@backstage/frontend-plugin-api';
import { slugPickerFormField } from './extensions';

export default createFrontendPlugin({
  pluginId: 'scaffolder-fields-acme',
  extensions: [slugPickerFormField],
});
```

For a plugin that exposes multiple fields, list them all in the `extensions` array.

### Adopter-side template usage

Once the adopter installs the plugin, templates reference the field by its `createFormField` name:

```yaml
parameters:
  - title: Component details
    properties:
      slug:
        type: string
        title: Slug
        description: URL-safe identifier
        ui:field: SlugPicker
        ui:options:
          allowUppercase: false
```

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin-module` and the user's intent describes scaffolder fields:

1. For each field in the intent, create a folder under `plugins/<id>/src/components/<FieldName>/` with `<FieldName>.tsx` (component), `schema.ts`, optionally `validation.ts`, and `index.ts` (the `createFormField` wrapper).
2. Create or extend `plugins/<id>/src/extensions.tsx` with one `FormFieldBlueprint.make({ name, params: { field: ... } })` per field. Use a kebab-case extension `name`; keep the user-facing `createFormField` name in PascalCase.
3. Add each blueprint extension to `plugins/<id>/src/plugin.tsx`'s `extensions` array.
4. Add `@backstage/plugin-scaffolder-react` (provides both the alpha and main entries) and `zod` to `dependencies`.

For a `frontend-plugin-module` template — the most common container for adopter field packages — the module's `pluginId` should target a value adopters know to bind (typically the adopter's umbrella plugin id, or the `scaffolder` plugin if extending the host directly). Confirm the user's intended adopter setup before settling.

## Tests

Test the building blocks independently — each has its own test file (component test in `references/field-component.md`, validation test in `references/field-validation.md`, schema test in `references/field-schema.md`). The `createFormField` and `FormFieldBlueprint.make` calls themselves are declarative and don't need bespoke tests beyond a smoke test that the plugin loads without throwing:

```ts
import { startTestApp } from '@backstage/frontend-test-utils';
import scaffolderFieldsAcme from './plugin';

it('plugin registers without error', async () => {
  await startTestApp({ features: [scaffolderFieldsAcme] });
});
```

(Verify `startTestApp`'s exact API against `../backstage/packages/frontend-test-utils` — for full integration tests, the canonical path is to render the scaffolder template form with the field installed and assert end-to-end behavior, but that's heavier than most field changes warrant.)
