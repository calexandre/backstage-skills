# Pattern: Register / refresh / unregister catalog locations

## Standalone usage — Frontend

### Register a new location

```tsx
import { useApi } from '@backstage/frontend-plugin-api';
import { catalogApiRef } from '@backstage/plugin-catalog-react';
import { Button } from '@backstage/ui';

const RegisterButton = ({ url }: { url: string }) => {
  const catalogApi = useApi(catalogApiRef);

  const onClick = async () => {
    const result = await catalogApi.addLocation({
      type: 'url',
      target: url,
    });
    console.log('registered entities:', result.entities);
  };

  return <Button onClick={onClick}>Register {url}</Button>;
};
```

### Refresh an entity

```ts
await catalogApi.refreshEntity('component:default/foo');
```

### Unregister a location

```ts
await catalogApi.removeLocationById(locationId);
```

To find the location id for an entity, read the `backstage.io/managed-by-location` annotation:

```ts
const entity = await catalogApi.getEntityByRef('component:default/foo');
const annotation =
  entity?.metadata.annotations?.['backstage.io/managed-by-origin-location'];
// annotation looks like 'url:https://github.com/...'
```

## Standalone usage — Backend

In a backend plugin or module, depend on `catalogServiceRef` from `@backstage/plugin-catalog-node`:

```ts
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';
import { catalogServiceRef } from '@backstage/plugin-catalog-node';

createBackendPlugin({
  pluginId: 'my-plugin',
  register(env) {
    env.registerInit({
      deps: {
        logger: coreServices.logger,
        auth: coreServices.auth,
        catalog: catalogServiceRef,
      },
      async init({ logger, auth, catalog }) {
        const credentials = await auth.getOwnServiceCredentials();

        // Register a new location
        const result = await catalog.addLocation(
          {
            type: 'url',
            target: 'https://github.com/foo/bar/blob/main/catalog-info.yaml',
          },
          { credentials },
        );
        logger.info(`registered ${result.entities.length} entities`);

        // Refresh an entity
        await catalog.refreshEntity('component:default/foo', { credentials });

        // Unregister a location by id
        // (The location id can be read from the `backstage.io/managed-by-location` annotation
        //  on any entity that location produced — split on ':' and take the second segment.)
        await catalog.removeLocationById('some-location-id', { credentials });
      },
    });
  },
});
```

Key points:

- Method names match the frontend API. Auth comes through `{ credentials }`, not `{ token }`.
- `addLocation` returns `{ entities: Entity[] }` describing what got registered.
- `refreshEntity` schedules a re-fetch — it doesn't block on the refresh actually completing.
- `removeLocationById` requires admin-level credentials in most setups; the service's own credentials usually work for plugins managed via app-config.
- For HTTP-triggered registration (e.g. an "import this URL" button calling your backend), swap `auth.getOwnServiceCredentials()` for `httpAuth.credentials(req, { allow: ['user'] })` so the operation runs as the requesting user.

## Post-scaffold usage — Frontend

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin`:

1. Locate the main page component.
2. Add a new file `plugins/<id>/src/components/RegisterLocation/RegisterLocation.tsx` containing the `RegisterButton` component above (rename to `RegisterLocation`).
3. Render `<RegisterLocation />` inside the page's `<Content>` block, with a placeholder `url` prop the user fills in.
4. Confirm `@backstage/plugin-catalog-react` is in dependencies.

## Post-scaffold usage — Backend

When invoked right after `creating-backstage-plugin` scaffolds a `backend-plugin` or `backend-plugin-module`:

1. Add `@backstage/plugin-catalog-node` to `dependencies`.
2. Add `catalog: catalogServiceRef` to deps. Add `httpAuth: coreServices.httpAuth` (for route handlers) and/or `auth: coreServices.auth` (for service-to-service) depending on context.
3. Wire the relevant call (`addLocation`, `refreshEntity`, or `removeLocationById`) into either an HTTP route handler (using `httpAuth.credentials(req)`) or a scheduled/init task (using `auth.getOwnServiceCredentials()`).

## Tests — Frontend

```tsx
import {
  TestApiProvider,
  renderInTestApp,
} from '@backstage/frontend-test-utils';
import { catalogApiRef } from '@backstage/plugin-catalog-react';
import { fireEvent, screen } from '@testing-library/react';
import { RegisterLocation } from './RegisterLocation';

const mockCatalogApi = {
  addLocation: jest.fn().mockResolvedValue({ entities: [] }),
} as any;

it('calls addLocation with the given url', async () => {
  renderInTestApp(
    <TestApiProvider apis={[[catalogApiRef, mockCatalogApi]]}>
      <RegisterLocation url="https://github.com/foo/bar/blob/main/catalog-info.yaml" />
    </TestApiProvider>,
  );
  fireEvent.click(await screen.findByText(/Register/));
  expect(mockCatalogApi.addLocation).toHaveBeenCalledWith({
    type: 'url',
    target: 'https://github.com/foo/bar/blob/main/catalog-info.yaml',
  });
});
```

## Tests — Backend

Use `catalogServiceMock.mock()` to get a service whose every method is a `jest.fn()` — ideal when you need to assert what was passed to `addLocation` / `refreshEntity` / `removeLocationById`:

```ts
import {
  startTestBackend,
  mockCredentials,
} from '@backstage/backend-test-utils';
import { catalogServiceMock } from '@backstage/plugin-catalog-node/testUtils';
import { myPlugin } from './plugin';

it('registers a location with the supplied url', async () => {
  const catalog = catalogServiceMock.mock();

  await startTestBackend({
    features: [myPlugin, catalog.factory],
  });

  expect(catalog.addLocation).toHaveBeenCalledWith(
    {
      type: 'url',
      target: 'https://github.com/foo/bar/blob/main/catalog-info.yaml',
    },
    { credentials: expect.any(Object) },
  );
});
```

For more direct unit-level tests of helpers that take a `CatalogService`:

```ts
const catalog = catalogServiceMock.mock();
catalog.addLocation.mockResolvedValue({
  entities: [],
  location: { id: 'loc-id', type: 'url', target: 'https://...' },
});

await myHelper({ catalog, credentials: mockCredentials.service() });
expect(catalog.addLocation).toHaveBeenCalled();
```
