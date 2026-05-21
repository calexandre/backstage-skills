# Pattern: Field component

The component is a regular React function component that receives `FieldExtensionComponentProps` from `@backstage/plugin-scaffolder-react`. It renders the input UI and reports value changes via `props.onChange`.

## Standalone usage

```tsx
import React from 'react';
import {
  FieldExtensionComponentProps,
  CustomFieldValidator,
} from '@backstage/plugin-scaffolder-react';
import { TextField, Text } from '@backstage/ui';

// The output type and ui:options shape come from your makeFieldSchema call.
// See references/field-schema.md.
export type SlugPickerProps = FieldExtensionComponentProps<
  string,
  { allowUppercase?: boolean }
>;

export const SlugPicker = (props: SlugPickerProps) => {
  const {
    onChange,
    required,
    schema: { title = 'Slug', description = 'A URL-safe slug' },
    formData,
    rawErrors,
    uiSchema,
    idSchema,
  } = props;

  const allowUppercase = uiSchema?.['ui:options']?.allowUppercase ?? false;
  const hasError = (rawErrors?.length ?? 0) > 0;

  return (
    <>
      <TextField
        id={idSchema?.$id}
        label={title}
        helperText={description}
        isRequired={required}
        value={formData ?? ''}
        onChange={value =>
          onChange(allowUppercase ? value : value.toLowerCase())
        }
      />
      {hasError && (
        <Text variant="body-small" color="danger">
          {rawErrors?.[0]}
        </Text>
      )}
    </>
  );
};
```

Key props on `FieldExtensionComponentProps<TValue, TUiOptions>`:

| Prop                    | What it gives you                                                                                               |
| ----------------------- | --------------------------------------------------------------------------------------------------------------- |
| `formData`              | The current value (typed as `TValue`) — use as the controlled input's value.                                    |
| `onChange(value)`       | Updates the form state. Call with the new value of type `TValue`.                                               |
| `required`              | Whether the template marked the field required (`*` in label).                                                  |
| `rawErrors`             | Array of error strings from validation (sync schema or async validation). Render below the input.               |
| `schema`                | The merged JSON schema for this field — `schema.title`, `schema.description`, `schema.default`, etc.            |
| `uiSchema`              | Template-supplied UI hints. Custom `ui:options` come through as `uiSchema['ui:options']` typed as `TUiOptions`. |
| `idSchema`              | RJSF id object. Use `idSchema.$id` for `<input id>` so labels work.                                             |
| `placeholder`           | Convenience pull-out from `uiSchema['ui:placeholder']`.                                                         |
| `disabled` / `readonly` | Set by the framework (e.g. when the form is being submitted). Forward to the input.                             |

The component should be a plain React component — it can use any hook (`useApi`, `useAsync`, `useJiraProject`, etc.) just like any other NFS component. It runs in the same React tree as the host app.

### Async data inside the field

For pickers that load options from the catalog or a backend API, use `useAsync` from `react-use`:

```tsx
import useAsync from 'react-use/lib/useAsync';
import { useApi } from '@backstage/frontend-plugin-api';
import { catalogApiRef } from '@backstage/plugin-catalog-react';
import { Select } from '@backstage/ui';

export const SystemPicker = (props: FieldExtensionComponentProps<string>) => {
  const catalogApi = useApi(catalogApiRef);

  const { value: items, loading } = useAsync(async () => {
    const { items } = await catalogApi.getEntities({
      filter: { kind: 'System' },
      fields: ['metadata.name', 'metadata.title'],
    });
    return items;
  }, [catalogApi]);

  return (
    <Select
      label={props.schema.title ?? 'System'}
      isDisabled={loading}
      selectedKey={props.formData}
      onSelectionChange={key => props.onChange(String(key))}
    >
      {(items ?? []).map(item => (
        <Select.Option key={item.metadata.name} id={item.metadata.name}>
          {item.metadata.title ?? item.metadata.name}
        </Select.Option>
      ))}
    </Select>
  );
};
```

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin-module` targeting the scaffolder:

1. Create `plugins/<id>/src/components/<FieldName>/<FieldName>.tsx` with a component matching the structure above.
2. Use BUI components (`TextField`, `Select`, `Skeleton`, `Text`, …) for the UI — never `@material-ui/core` unless the field needs something BUI doesn't have.
3. The `FieldExtensionComponentProps` generic args must match the schema declared in `references/field-schema.md` — tie them together via a shared type alias.
4. Add `@backstage/plugin-scaffolder-react` and `@backstage/ui` to `package.json` dependencies.

## Tests

Field components are plain React components — render them with plain `render` from `@testing-library/react` (no test app needed unless the component itself uses `useApi` / `useRouteRef`):

```tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { SlugPicker } from './SlugPicker';

const baseProps = {
  onChange: jest.fn(),
  required: false,
  rawErrors: [],
  schema: { title: 'Slug', description: 'A URL-safe slug' },
  uiSchema: {},
  idSchema: { $id: 'root_slug' },
  formData: '',
} as any;

beforeEach(() => baseProps.onChange.mockReset());

it('lowercases input by default', () => {
  render(<SlugPicker {...baseProps} />);
  fireEvent.change(screen.getByLabelText('Slug'), {
    target: { value: 'My Slug' },
  });
  expect(baseProps.onChange).toHaveBeenCalledWith('my slug');
});

it('preserves case when allowUppercase is on', () => {
  render(
    <SlugPicker
      {...baseProps}
      uiSchema={{ 'ui:options': { allowUppercase: true } }}
    />,
  );
  fireEvent.change(screen.getByLabelText('Slug'), {
    target: { value: 'My Slug' },
  });
  expect(baseProps.onChange).toHaveBeenCalledWith('My Slug');
});

it('renders rawErrors below the input', () => {
  render(<SlugPicker {...baseProps} rawErrors={['must be unique']} />);
  expect(screen.getByText('must be unique')).toBeInTheDocument();
});
```

For components that use `useApi`, wrap with `TestApiProvider` from `@backstage/frontend-test-utils` and pass mocked APIs — same pattern as the catalog/identity skill tests.
