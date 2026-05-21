# Pattern: Query entities with filters

## Standalone usage — Frontend

```tsx
import React from 'react';
import useAsync from 'react-use/lib/useAsync';
import { useApi } from '@backstage/frontend-plugin-api';
import { catalogApiRef } from '@backstage/plugin-catalog-react';
import { Skeleton, Alert, Flex, Text } from '@backstage/ui';

export const TeamGroups = () => {
  const catalogApi = useApi(catalogApiRef);

  const { value, loading, error } = useAsync(async () => {
    const { items } = await catalogApi.getEntities({
      filter: { kind: 'Group', 'spec.type': 'team' },
      fields: ['metadata.name', 'metadata.title', 'spec.type'],
    });
    return items;
  }, [catalogApi]);

  if (loading) return <Skeleton width={240} height={120} />;
  if (error) return <Alert status="danger" title={`Error: ${error.message}`} />;

  return (
    <Flex direction="column" gap="2">
      {value?.map(g => (
        <Text key={g.metadata.name}>{g.metadata.title ?? g.metadata.name}</Text>
      ))}
    </Flex>
  );
};
```

Key points:

- `filter` is AND-combined within one object. Pass an array of objects for OR.
- `fields` is optional — use it only when needed to limit the payload (e.g. fetching a large number of entities where you only read a few fields).
- For paginated results in larger catalogs, prefer `queryEntities()` over `getEntities()` (it returns a cursor).

### OR example

```ts
filter: [
  { kind: 'Component', 'spec.type': 'service' },
  { kind: 'Component', 'spec.type': 'website' },
];
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
        scheduler: coreServices.scheduler,
        auth: coreServices.auth,
        catalog: catalogServiceRef,
      },
      async init({ logger, scheduler, auth, catalog }) {
        await scheduler.scheduleTask({
          id: 'sync-team-groups',
          frequency: { hours: 1 },
          timeout: { minutes: 10 },
          fn: async () => {
            const credentials = await auth.getOwnServiceCredentials();

            const { items } = await catalog.getEntities(
              {
                filter: { kind: 'Group', 'spec.type': 'team' },
                fields: ['metadata.name', 'metadata.title', 'spec.type'],
              },
              { credentials },
            );

            logger.info(`synced ${items.length} team groups`);
          },
        });
      },
    });
  },
});
```

For HTTP route handlers, swap the `auth.getOwnServiceCredentials()` call for `httpAuth.credentials(req, { allow: ['user'] })` to query as the requesting user.

Key points:

- `getEntities` returns `{ items: Entity[] }` (same shape as the frontend API).
- Filters and fields are identical — `filter` is AND within an object, OR across an array; `fields` is optional and used to trim the payload.
- For very large result sets, use `queryEntities` (cursor-based pagination) instead of `getEntities`.

## Post-scaffold usage — Frontend

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin`:

1. Locate the main page component (typically `plugins/<id>/src/components/ExampleComponent/ExampleComponent.tsx`).
2. Add the standalone snippet as a new component file: `plugins/<id>/src/components/TeamGroups/TeamGroups.tsx`.
3. Render `<TeamGroups />` inside the page's `<Content>` block.
4. Add `@backstage/plugin-catalog-react` to the plugin's `package.json` dependencies if not already present.
5. Run `yarn install` from the repo root.

## Post-scaffold usage — Backend

When invoked right after `creating-backstage-plugin` scaffolds a `backend-plugin` or `backend-plugin-module`:

1. Add `@backstage/plugin-catalog-node` to `dependencies`.
2. Add `catalog: catalogServiceRef` to deps. Also add `httpAuth: coreServices.httpAuth` (for route handlers) and/or `auth: coreServices.auth` (for scheduled tasks / init-time fetches) depending on where queries happen.
3. Wrap the catalog query in either an HTTP route handler (using `httpAuth.credentials(req)`) or a scheduled task (using `auth.getOwnServiceCredentials()`).

## Tests — Frontend

```tsx
import {
  TestApiProvider,
  renderInTestApp,
} from '@backstage/frontend-test-utils';
import { catalogApiRef } from '@backstage/plugin-catalog-react';
import { screen } from '@testing-library/react';
import { TeamGroups } from './TeamGroups';

const mockCatalogApi = {
  getEntities: jest.fn().mockResolvedValue({
    items: [
      {
        apiVersion: 'backstage.io/v1alpha1',
        kind: 'Group',
        metadata: { name: 'team-a', title: 'Team A' },
        spec: { type: 'team' },
      },
      {
        apiVersion: 'backstage.io/v1alpha1',
        kind: 'Group',
        metadata: { name: 'team-b' },
        spec: { type: 'team' },
      },
    ],
  }),
} as any;

it('renders all team groups', async () => {
  renderInTestApp(
    <TestApiProvider apis={[[catalogApiRef, mockCatalogApi]]}>
      <TeamGroups />
    </TestApiProvider>,
  );
  expect(await screen.findByText('Team A')).toBeInTheDocument();
  expect(await screen.findByText('team-b')).toBeInTheDocument();
});
```

## Tests — Backend

Use `catalogServiceMock` from `@backstage/plugin-catalog-node/testUtils` to install a fake catalog service:

```ts
import {
  startTestBackend,
  mockCredentials,
} from '@backstage/backend-test-utils';
import { catalogServiceMock } from '@backstage/plugin-catalog-node/testUtils';
import { myPlugin } from './plugin';

it('schedules a sync that queries team groups', async () => {
  await startTestBackend({
    features: [
      myPlugin,
      catalogServiceMock.factory({
        entities: [
          {
            apiVersion: 'backstage.io/v1alpha1',
            kind: 'Group',
            metadata: { name: 'team-a', title: 'Team A' },
            spec: { type: 'team' },
          },
        ],
      }),
    ],
  });

  // Assert via your own observable side effects (logs, db rows, etc.).
});
```

For unit tests of helpers that take a `CatalogService` directly, pass the service from `catalogServiceMock(...)` (the in-memory variant) plus `mockCredentials.service()`:

```ts
const catalog = catalogServiceMock({ entities: [...] });
const { items } = await catalog.getEntities(
  { filter: { kind: 'Group' } },
  { credentials: mockCredentials.service() },
);
```

Use `catalogServiceMock.mock()` when you need `jest.fn()` per method to assert call arguments individually.
