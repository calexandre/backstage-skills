# Pattern: Backend authorization

In a backend plugin's route handler (or any other place an action is performed), check `coreServices.permissions.authorize(...)` before doing the work. The frontend gate is for UX — the backend gate is the actual access control. **Always do both.**

## Standalone usage

```ts
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';
import { AuthorizeResult } from '@backstage/plugin-permission-common';
import { NotAllowedError } from '@backstage/errors';
import express from 'express';
import Router from 'express-promise-router';
import { z } from 'zod';
import {
  incidentsCreatePermission,
  incidentsDeletePermission,
} from '@your/plugin-incidents-common';

createBackendPlugin({
  pluginId: 'incidents',
  register(env) {
    env.registerInit({
      deps: {
        logger: coreServices.logger,
        httpRouter: coreServices.httpRouter,
        httpAuth: coreServices.httpAuth,
        permissions: coreServices.permissions,
      },
      async init({ logger, httpRouter, httpAuth, permissions }) {
        const router = Router();
        router.use(express.json());

        router.post('/incidents', async (req, res) => {
          const credentials = await httpAuth.credentials(req, {
            allow: ['user'],
          });

          const [decision] = await permissions.authorize(
            [{ permission: incidentsCreatePermission }],
            { credentials },
          );
          if (decision.result !== AuthorizeResult.ALLOW) {
            throw new NotAllowedError(
              'caller is not allowed to create incidents',
            );
          }

          const body = z.object({ title: z.string() }).parse(req.body);
          logger.info(`creating incident ${body.title}`);
          // ... do the work ...
          res.status(201).json({ id: 'generated-id' });
        });

        router.delete('/incidents/:id', async (req, res) => {
          const credentials = await httpAuth.credentials(req, {
            allow: ['user'],
          });

          const [decision] = await permissions.authorize(
            [{ permission: incidentsDeletePermission }],
            { credentials },
          );
          if (decision.result !== AuthorizeResult.ALLOW) {
            throw new NotAllowedError('not allowed to delete incidents');
          }

          // ... do the work ...
          res.status(204).end();
        });

        httpRouter.use(router);
      },
    });
  },
});
```

Key points:

- **`permissions.authorize` returns an array** matching the order of requests. For a single permission, destructure `const [decision] = await permissions.authorize(...)`.
- **`decision.result`** is an `AuthorizeResult` enum: `ALLOW`, `DENY`, or `CONDITIONAL`. For basic (non-resource) permissions, `CONDITIONAL` doesn't appear — only `ALLOW`/`DENY`.
- **Throw `NotAllowedError`** (HTTP 403) when denied — see `using-backstage-backend-services/references/http-router.md` for the typed errors convention. Don't `res.status(403).send(...)`.
- **Always read credentials first**, then authorize, then do the work. The `httpAuth.credentials(req)` call validates the token; the `permissions.authorize` call checks whether the validated principal can perform the action.

### Batching authorize calls

When the same handler needs to check multiple permissions, batch them into one `authorize` call:

```ts
const [createDecision, deleteDecision] = await permissions.authorize(
  [
    { permission: incidentsCreatePermission },
    { permission: incidentsDeletePermission },
  ],
  { credentials },
);
```

Reduces round-trips and gives the policy implementation a chance to optimize across the requests.

### Service-to-service callers

When the route is meant to be called by another plugin (not directly by users), allow service principals too:

```ts
const credentials = await httpAuth.credentials(req, {
  allow: ['user', 'service'],
});
```

Service-to-service callers usually skip permission checks (they're trusted infrastructure). Decide whether to call `permissions.authorize` based on the principal type: `credentials.principal.type === 'service'` → skip; `'user'` → check.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `backend-plugin` AND intent describes access control:

1. Depend on the plugin's `*-common` package (which exports the permission objects from `references/declaring.md`). For a standalone plugin, import from `'../permissions'`.
2. Add `permissions: coreServices.permissions` to the deps map. Keep `httpAuth: coreServices.httpAuth` for reading the caller's credentials.
3. Inside each route handler that performs a guarded action, add the `await permissions.authorize([...], { credentials })` check between the `httpAuth.credentials(req)` call and the actual work.
4. Throw `NotAllowedError` from `@backstage/errors` when denied.
5. Add `@backstage/plugin-permission-common`, `@backstage/errors`, and the plugin's `*-common` package to `dependencies`.

## Tests

Use `mockServices.permissions(...)` from `@backstage/backend-test-utils` to install a mock permissions service. The mock supports per-permission allow/deny and a default fallback:

```ts
import {
  startTestBackend,
  mockServices,
  mockCredentials,
} from '@backstage/backend-test-utils';
import { AuthorizeResult } from '@backstage/plugin-permission-common';
import request from 'supertest';
import { incidentsCreatePermission } from '@your/plugin-incidents-common';
import { incidentsPlugin } from './plugin';

it('creates an incident when allowed', async () => {
  const { server } = await startTestBackend({
    features: [
      incidentsPlugin,
      mockServices.permissions.factory({
        permissions: [
          {
            permission: incidentsCreatePermission,
            result: AuthorizeResult.ALLOW,
          },
        ],
      }),
      mockServices.httpAuth.factory({
        defaultCredentials: mockCredentials.user(),
      }),
    ],
  });

  const res = await request(server)
    .post('/api/incidents/incidents')
    .set('Authorization', mockCredentials.user.header())
    .send({ title: 'Outage' });

  expect(res.status).toBe(201);
});

it('returns 403 when not allowed', async () => {
  const { server } = await startTestBackend({
    features: [
      incidentsPlugin,
      mockServices.permissions.factory({
        permissions: [
          {
            permission: incidentsCreatePermission,
            result: AuthorizeResult.DENY,
          },
        ],
      }),
      mockServices.httpAuth.factory({
        defaultCredentials: mockCredentials.user(),
      }),
    ],
  });

  const res = await request(server)
    .post('/api/incidents/incidents')
    .set('Authorization', mockCredentials.user.header())
    .send({ title: 'Outage' });

  expect(res.status).toBe(403);
});
```

Verify the exact `mockServices.permissions` factory shape against `../backstage/packages/backend-test-utils/src/services/MockPermissionsService.ts` — the shape of the `permissions` array, the default fallback behavior, and any `mockServices.permissions.mock()` variant.
