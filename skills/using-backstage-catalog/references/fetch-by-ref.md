# Pattern: Fetch one entity by ref

## Standalone usage — Frontend

```tsx
import React from 'react';
import useAsync from 'react-use/lib/useAsync';
import { useApi } from '@backstage/frontend-plugin-api';
import { catalogApiRef } from '@backstage/plugin-catalog-react';
import { Skeleton, Alert, Text } from '@backstage/ui';

type Props = { entityRef: string };

export const EntityName = ({ entityRef }: Props) => {
  const catalogApi = useApi(catalogApiRef);

  const {
    value: entity,
    loading,
    error,
  } = useAsync(
    () => catalogApi.getEntityByRef(entityRef),
    [catalogApi, entityRef],
  );

  if (loading) return <Skeleton width={160} height={20} />;
  if (error) return <Alert status="danger" title={`Error: ${error.message}`} />;
  if (!entity) return <Alert status="info" title="Entity not found" />;

  return <Text>{entity.metadata.title ?? entity.metadata.name}</Text>;
};
```

Key points:

- Pass either a string ref (`'group:default/team-a'`) or a `CompoundEntityRef` object.
- Always handle the `undefined` return — entities can be deleted between fetches.

## Standalone usage — Backend

In a backend plugin or module, depend on `catalogServiceRef` from `@backstage/plugin-catalog-node`:

```ts
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';
import { catalogServiceRef } from '@backstage/plugin-catalog-node';
import { NotFoundError } from '@backstage/errors';
import express from 'express';
import Router from 'express-promise-router';

createBackendPlugin({
  pluginId: 'my-plugin',
  register(env) {
    env.registerInit({
      deps: {
        logger: coreServices.logger,
        httpRouter: coreServices.httpRouter,
        httpAuth: coreServices.httpAuth,
        catalog: catalogServiceRef,
      },
      async init({ logger, httpRouter, httpAuth, catalog }) {
        const router = Router();

        router.get('/components/:name', async (req, res) => {
          const credentials = await httpAuth.credentials(req, {
            allow: ['user', 'service'],
          });

          const entity = await catalog.getEntityByRef(
            `component:default/${req.params.name}`,
            { credentials },
          );

          if (!entity) {
            throw new NotFoundError(`component ${req.params.name} not found`);
          }

          res.json({ name: entity.metadata.name, kind: entity.kind });
        });

        httpRouter.use(router);
      },
    });
  },
});
```

For service-to-service contexts (scheduled tasks, init-time work — no incoming request):

```ts
deps: {
  auth: coreServices.auth,
  catalog: catalogServiceRef,
},
async init({ auth, catalog }) {
  const credentials = await auth.getOwnServiceCredentials();
  const entity = await catalog.getEntityByRef('component:default/foo', {
    credentials,
  });
}
```

Key points:

- `catalogServiceRef` already has `coreServices.auth` and `coreServices.discovery` wired in internally — you just inject `catalog` and call its methods. Don't manually `new CatalogClient(...)`.
- Methods take `{ credentials }` (a `BackstageCredentials` object), NOT `{ token }`. Get credentials via `httpAuth.credentials(req)` (incoming requests) or `auth.getOwnServiceCredentials()` (service-to-service).
- Returns `Entity | undefined` — same shape as the frontend API.
- Same `Entity` interface, same `getEntityByRef` semantics — only the auth surface differs from `CatalogClient`.

## Post-scaffold usage — Frontend

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin`:

1. Locate the main page component (typically `plugins/<id>/src/components/ExampleComponent/ExampleComponent.tsx`).
2. Add the standalone snippet as a child component file: `plugins/<id>/src/components/EntityName/EntityName.tsx`.
3. Wire it into the page with a hardcoded sample ref for initial demo (e.g. `<EntityName entityRef="component:default/example-component" />`); the user replaces with real data later.
4. Add `@backstage/plugin-catalog-react` and `@backstage/catalog-model` to the plugin's `package.json` dependencies if not already present.
5. Run `yarn install` from the repo root to pick up new deps.

## Post-scaffold usage — Backend

When invoked right after `creating-backstage-plugin` scaffolds a `backend-plugin` or `backend-plugin-module`:

1. Add `@backstage/plugin-catalog-node` and `@backstage/errors` to the plugin's `package.json` dependencies.
2. In `src/plugin.ts` (or `src/module.ts`), add `catalog: catalogServiceRef` to the deps map. Also add `httpAuth: coreServices.httpAuth` (for HTTP routes) and/or `auth: coreServices.auth` (for service-to-service) depending on where catalog reads happen.
3. Inside `init`, call `catalog.getEntityByRef(ref, { credentials })` — passing credentials from `httpAuth.credentials(req)` in route handlers, or `auth.getOwnServiceCredentials()` in scheduled tasks / init-time fetches.
4. For "not found" cases, throw `new NotFoundError(...)` from `@backstage/errors` — don't manually return a 404 status.
5. Run `yarn install` from the repo root.

## Tests — Frontend

```tsx
import {
  TestApiProvider,
  renderInTestApp,
} from '@backstage/frontend-test-utils';
import { catalogApiRef } from '@backstage/plugin-catalog-react';
import { screen } from '@testing-library/react';
import { EntityName } from './EntityName';

const mockCatalogApi = {
  getEntityByRef: jest.fn().mockResolvedValue({
    apiVersion: 'backstage.io/v1alpha1',
    kind: 'Component',
    metadata: { name: 'example', title: 'Example' },
    spec: {},
  }),
} as any;

it('renders the entity title', async () => {
  renderInTestApp(
    <TestApiProvider apis={[[catalogApiRef, mockCatalogApi]]}>
      <EntityName entityRef="component:default/example" />
    </TestApiProvider>,
  );
  expect(await screen.findByText('Example')).toBeInTheDocument();
});

it('shows not-found when the entity is missing', async () => {
  const api = { getEntityByRef: jest.fn().mockResolvedValue(undefined) } as any;
  renderInTestApp(
    <TestApiProvider apis={[[catalogApiRef, api]]}>
      <EntityName entityRef="component:default/missing" />
    </TestApiProvider>,
  );
  expect(await screen.findByText('Entity not found')).toBeInTheDocument();
});
```

## Tests — Backend

In `startTestBackend`, install a `catalogServiceMock` to substitute the catalog service. The shape mirrors `CatalogService` — provide jest mocks for the methods you call:

```ts
import {
  startTestBackend,
  mockServices,
  mockCredentials,
} from '@backstage/backend-test-utils';
import { catalogServiceMock } from '@backstage/plugin-catalog-node/testUtils';
import request from 'supertest';
import { myPlugin } from './plugin';

it('returns the component name', async () => {
  const { server } = await startTestBackend({
    features: [
      myPlugin,
      catalogServiceMock.factory({
        entities: [
          {
            apiVersion: 'backstage.io/v1alpha1',
            kind: 'Component',
            metadata: { name: 'example' },
            spec: {},
          },
        ],
      }),
      mockServices.httpAuth.factory({
        defaultCredentials: mockCredentials.user(),
      }),
    ],
  });

  const response = await request(server)
    .get('/api/my-plugin/components/example')
    .set('Authorization', mockCredentials.user.header());

  expect(response.status).toBe(200);
  expect(response.body.name).toBe('example');
});
```

`catalogServiceMock` (from `@backstage/plugin-catalog-node/testUtils`) has three forms:

- `catalogServiceMock(options)` — returns an in-memory `CatalogServiceMock` for direct use in unit tests.
- `catalogServiceMock.factory(options)` — returns a `ServiceFactory` for `startTestBackend({ features })`.
- `catalogServiceMock.mock()` — returns a service whose every method is a `jest.fn()`, for assertion-heavy tests where you need to control individual return values.

For unit tests of helpers that take a `CatalogService` directly, pass a hand-rolled object with the methods you call:

```ts
const catalog = {
  getEntityByRef: jest.fn().mockResolvedValue(undefined),
} as any;

await myHelper({ catalog, credentials: mockCredentials.service() });
```
