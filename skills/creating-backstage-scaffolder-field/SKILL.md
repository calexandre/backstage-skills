---
name: creating-backstage-scaffolder-field
description: Use when building a custom scaffolder form field — a React component that scaffolder template forms can use via `ui:field: <name>`. Covers FormFieldBlueprint, createFormField, the field component props (formData, onChange, schema, rawErrors, etc.), Zod-based field schema via makeFieldSchema, optional validation, and registering the field as a frontend extension. Triggers on phrases like "scaffolder field", "custom field extension", "form field for scaffolder templates", "ui:field".
---

# Creating a Backstage Scaffolder Form Field

## When to use this skill

- Building a custom React component that scaffolder template forms can use via `ui:field: '<YourName>'`
- Editing an existing scaffolder field extension
- The user wants something the built-in fields (`EntityNamePicker`, `EntityPicker`, `OwnerPicker`, `RepoUrlPicker`, etc.) don't cover — e.g. a picker that queries a domain-specific external system, a constrained text input with custom validation, a multi-step composite field

## What you build

A scaffolder field is a React component plus three pieces of metadata (name, schema, optional validation) wrapped together into a `FormField` via `createFormField`. The `FormField` is then registered as a plugin extension via `FormFieldBlueprint.make`, listed in the plugin's `extensions` array.

Three-layer structure:

1. **The component** — a React component receiving `FieldExtensionComponentProps<TValue, TUiOptions>` and rendering the input UI.
2. **The schema** — a `makeFieldSchema({ output, uiOptions? })` Zod schema declaring the field's stored value type and optional UI options.
3. **The blueprint extension** — `FormFieldBlueprint.make({ name, params: { field: () => Promise<FormField> } })` so the field is discovered by the scaffolder and exposed under the name templates reference in `ui:field`.

For an adopter, the typical container is a `frontend-plugin-module` that targets the scaffolder, scaffolded via `yarn new --select frontend-plugin-module`. A standalone frontend plugin can also expose fields via the same blueprint.

## Choose the right pattern

| Building...                                                     | Reference                        |
| --------------------------------------------------------------- | -------------------------------- |
| The field's React component                                     | references/field-component.md    |
| The field's Zod schema (`makeFieldSchema`)                      | references/field-schema.md       |
| Optional validation function                                    | references/field-validation.md   |
| Plugin/module wiring (`createFormField` + `FormFieldBlueprint`) | references/field-registration.md |

## Two invocation modes

- **Standalone**: editing an existing scaffolder-field plugin or module.
- **Post-scaffold** (called by `creating-backstage-plugin` for `frontend-plugin-module` templates targeting `scaffolder`): scaffold the component + schema + registration together.

## Common gotchas

- The blueprint and helper come from `@backstage/plugin-scaffolder-react/alpha`, not the main entry. The package is published with the API marked `@alpha`; track the import path against `../backstage/plugins/scaffolder-react/src/next/blueprints/FormFieldBlueprint.tsx`.
- The field component receives RJSF (`@rjsf/utils`) props via `FieldExtensionComponentProps`. Use `props.onChange(newValue)` to update form state; don't call `setState` on parents.
- `makeFieldSchema({ output: z => z.string(), uiOptions: z => z.object({ ... }) })` produces both the runtime schema and a `TProps` type. Always declare `output`; `uiOptions` is optional but useful when templates pass field-specific config via `ui:options`.
- Templates reference the field by the `name` you gave to `createFormField` and `FormFieldBlueprint.make` — keep them identical. Use PascalCase (`MyCoolPicker`); templates write `ui:field: 'MyCoolPicker'`.
- Validation runs after the user submits — for synchronous in-input validation, use the schema's Zod constraints. The `validation` function is best for cross-field checks or async-resolved errors that come back from an external API.
- Field components can call `useApi(...)` for catalog/identity/etc. just like any other NFS component — they live in the same React tree as the rest of the app.
