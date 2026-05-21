# Pattern: Field validation

Validation runs after the user submits the form. Use it for checks the schema can't express — async lookups, cross-field constraints, formats that depend on app config.

For purely-syntactic constraints (regex, min/max length, required), prefer the Zod schema in `references/field-schema.md` — those produce inline errors without a roundtrip.

## Standalone usage

### Sync validation

```ts
// src/components/SlugPicker/validation.ts
import { CustomFieldValidator } from '@backstage/plugin-scaffolder-react';

export const slugPickerValidation: CustomFieldValidator<string> = (
  value,
  validation,
) => {
  if (!value) {
    // Don't add an error for empty when the field isn't required —
    // the framework handles required-ness from the schema.
    return;
  }

  if (!/^[a-z0-9-]+$/.test(value)) {
    validation.addError(
      'Slug must contain only lowercase letters, digits, and dashes.',
    );
  }

  if (value.startsWith('-') || value.endsWith('-')) {
    validation.addError('Slug must not start or end with a dash.');
  }
};
```

`validation.addError(message)` adds an entry to the field's `rawErrors` array. The error renders below the input via the component's `rawErrors` prop (see `references/field-component.md`).

### Async validation

For checks that need to call out (catalog lookup, backend availability, uniqueness), the validator can be async. The third argument carries `apiHolder` for resolving APIs:

```ts
import { CustomFieldValidator } from '@backstage/plugin-scaffolder-react';
import { catalogApiRef } from '@backstage/plugin-catalog-react';

export const uniqueComponentNameValidation: CustomFieldValidator<
  string
> = async (value, validation, context) => {
  if (!value) return;

  const catalogApi = context.apiHolder.get(catalogApiRef);
  if (!catalogApi) {
    // API not available in this app — skip the check rather than fail-closed.
    return;
  }

  const existing = await catalogApi.getEntityByRef(
    `component:default/${value}`,
  );
  if (existing) {
    validation.addError(`A component named "${value}" already exists.`);
  }
};
```

Verify the exact `context` shape against `../backstage/plugins/scaffolder-react/src/extensions/types.ts` — older versions named the API holder differently.

### When to skip the validation function entirely

- The field's only constraint is "not empty" — let the schema's `required` flag handle it.
- The field's only constraint is a Zod-expressible shape — bake it into the `output` schema instead. Errors there appear at the same place in the UI.
- The field is read-only / display-only — no validation needed.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin-module`:

1. Create `plugins/<id>/src/components/<FieldName>/validation.ts` only when the user's intent describes constraints beyond what Zod alone covers. Otherwise skip — fewer files is better.
2. Use the typed `CustomFieldValidator<TValue, TUiOptions>` import to keep the value type in sync with the schema.
3. For async validators, fail open (`return` without adding an error) when the required API isn't available, rather than blocking submission.

## Tests

Validators are pure functions — call them directly with a stub validation object:

```ts
import { slugPickerValidation } from './validation';

const makeValidation = () => {
  const errors: string[] = [];
  return {
    addError: (msg: string) => errors.push(msg),
    errors,
  };
};

it('accepts a valid slug', () => {
  const v = makeValidation();
  slugPickerValidation('my-slug', v as any, {} as any);
  expect(v.errors).toEqual([]);
});

it('rejects uppercase letters', () => {
  const v = makeValidation();
  slugPickerValidation('My-Slug', v as any, {} as any);
  expect(v.errors).toContain(
    'Slug must contain only lowercase letters, digits, and dashes.',
  );
});

it('rejects leading or trailing dashes', () => {
  const v = makeValidation();
  slugPickerValidation('-foo', v as any, {} as any);
  expect(v.errors).toContain('Slug must not start or end with a dash.');
});
```

For async validators that read APIs via `context.apiHolder`, build a minimal `apiHolder` stub:

```ts
import { uniqueComponentNameValidation } from './validation';
import { catalogApiRef } from '@backstage/plugin-catalog-react';

it('rejects names that already exist', async () => {
  const v = makeValidation();
  const apiHolder = {
    get: (ref: any) =>
      ref === catalogApiRef
        ? { getEntityByRef: jest.fn().mockResolvedValue({ kind: 'Component' }) }
        : undefined,
  };

  await uniqueComponentNameValidation('foo', v as any, { apiHolder } as any);
  expect(v.errors).toContain('A component named "foo" already exists.');
});
```
