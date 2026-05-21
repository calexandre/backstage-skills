# Pattern: NFS hooks (`useApi`, `useRouteRef`)

In the new frontend system, hooks are imported from `@backstage/frontend-plugin-api` rather than `@backstage/core-plugin-api`. The hook signatures are unchanged for `useApi`; `useRouteRef` has one important behavioral change for external route refs.

## Standalone usage

### `useApi`

Identical signature to the old system, just a different import path:

```ts
import {
  useApi,
  identityApiRef,
  configApiRef,
} from '@backstage/frontend-plugin-api';

const identityApi = useApi(identityApiRef);
const config = useApi(configApiRef);
```

For plugin-specific API refs (e.g. `catalogApiRef` from `@backstage/plugin-catalog-react`), keep the import from the plugin package — those refs aren't re-exported from `@backstage/frontend-plugin-api`.

### `useRouteRef` — own routes

```ts
import { useRouteRef } from '@backstage/frontend-plugin-api';
import { detailsRouteRef } from '../routes';

const link = useRouteRef(detailsRouteRef);
const url = link({ id: 'foo' });
```

`useRouteRef` for a route the plugin owns always returns a function (the route is always present).

### `useRouteRef` — external routes

For external route refs, the return value can be `undefined` if the external route isn't bound (i.e. the target plugin isn't installed):

```ts
import { useRouteRef } from '@backstage/frontend-plugin-api';
import { externalDocsRouteRef } from '../routes';

export function MaybeDocsLink() {
  const docsLink = useRouteRef(externalDocsRouteRef);

  if (!docsLink) {
    return null; // external route not bound
  }

  return <a href={docsLink()}>Docs</a>;
}
```

This is a behavioral change from the old system, where `useRouteRef` for a non-bound external ref would throw. Always handle the `undefined` case.

### Common API refs available from `@backstage/frontend-plugin-api`

| Ref               | Use                                                    |
| ----------------- | ------------------------------------------------------ |
| `configApiRef`    | Read `app-config.yaml` config values                   |
| `discoveryApiRef` | Resolve the base URL of a backend plugin               |
| `fetchApiRef`     | Authenticated fetch (use instead of global `fetch`)    |
| `identityApiRef`  | Current user identity (see `using-backstage-identity`) |
| `storageApiRef`   | Persistent client-side storage                         |
| `analyticsApiRef` | Track events                                           |
| `errorApiRef`     | Surface errors to the global error display             |

For plugin-specific refs, import them from the plugin's own package (e.g. `catalogApiRef` from `@backstage/plugin-catalog-react`).

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a frontend template, ensure all hook imports use the NFS package:

1. Replace `from '@backstage/core-plugin-api'` with `from '@backstage/frontend-plugin-api'` for any of: `useApi`, `useRouteRef`, `configApiRef`, `discoveryApiRef`, `fetchApiRef`, `identityApiRef`, `storageApiRef`, `analyticsApiRef`, `errorApiRef`, `createApiRef`.
2. Leave plugin-specific refs alone (e.g. `catalogApiRef` continues to come from `@backstage/plugin-catalog-react`).
3. Add `@backstage/frontend-plugin-api` to `package.json` dependencies if not already present; remove `@backstage/core-plugin-api` if no longer used in the file.

## Tests

Tests provide mock implementations under their refs via `TestApiProvider` from `@backstage/frontend-test-utils`. For built-in core APIs, prefer the `mockApis.*` factories over hand-rolled mocks:

```tsx
import {
  TestApiProvider,
  mockApis,
  renderInTestApp,
} from '@backstage/frontend-test-utils';
import { configApiRef } from '@backstage/frontend-plugin-api';

it('reads config', async () => {
  renderInTestApp(
    <TestApiProvider
      apis={[
        [configApiRef, mockApis.config({ data: { my: { key: 'value' } } })],
      ]}
    >
      <MyComponent />
    </TestApiProvider>,
  );
});
```

For plugin-specific APIs without a `mockApis` factory, pass a hand-rolled object:

```tsx
const myMock = { fetchSomething: jest.fn().mockResolvedValue({ id: 'foo' }) };

renderInTestApp(
  <TestApiProvider apis={[[myPluginApiRef, myMock]]}>
    <MyComponent />
  </TestApiProvider>,
);
```

For `useRouteRef` mocking, prefer integration-level tests over unit tests — the routing wiring lives at the extension level. If you must unit-test, mock the entire `@backstage/frontend-plugin-api` module's `useRouteRef` export and return a plain function.
