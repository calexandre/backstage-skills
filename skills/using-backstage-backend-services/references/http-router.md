# Pattern: HTTP router

Routes are registered via `coreServices.httpRouter`. The plugin-scoped router automatically mounts all routes under `/api/<pluginId>/`.

## Standalone usage

```ts
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';
import { InputError, NotFoundError } from '@backstage/errors';
import express from 'express';
import Router from 'express-promise-router';
import { z } from 'zod';

createBackendPlugin({
  pluginId: 'my-plugin',
  register(env) {
    env.registerInit({
      deps: {
        logger: coreServices.logger,
        httpRouter: coreServices.httpRouter,
      },
      async init({ logger, httpRouter }) {
        const router = Router();
        router.use(express.json());

        router.get('/health', (_req, res) => {
          res.json({ ok: true });
        });

        router.get('/items/:id', async (req, res) => {
          const item = await loadItem(req.params.id);
          if (!item) {
            throw new NotFoundError(`item ${req.params.id} not found`);
          }
          res.json(item);
        });

        router.post('/items', async (req, res) => {
          const parsed = z.object({ name: z.string() }).safeParse(req.body);
          if (!parsed.success) {
            throw new InputError(`invalid body: ${parsed.error.message}`);
          }
          logger.info(`creating item ${parsed.data.name}`);
          res.status(201).json({ id: 'generated-id' });
        });

        httpRouter.use(router);
      },
    });
  },
});
```

Routes mount at:

- `GET /api/my-plugin/health`
- `GET /api/my-plugin/items/:id`
- `POST /api/my-plugin/items`

Key points:

- Express version: 4.x (Backstage 1.50). Standard Express middleware works as-is.
- Use `express-promise-router` (preferred over `express.Router()`) so async route handlers' rejected promises propagate to the error middleware automatically. The default scaffold already uses it.
- `httpRouter.use(router)` is the only hook — there's no per-method shorthand. Build a normal router and pass it in.
- All routes by default require a Backstage user or service token. To make a route public, use `httpRouter.addAuthPolicy({ path, allow })`:

```ts
httpRouter.addAuthPolicy({ path: '/health', allow: 'unauthenticated' });
```

Allowed values: `'unauthenticated'` (anyone), `'user-cookie'` (browser session only), `'user'`, `'service'`.

For request authentication and reading credentials inside a handler, see `references/auth.md` (`coreServices.httpAuth`).

### Throwing typed errors

For non-2xx responses, throw a typed error from `@backstage/errors` rather than calling `res.status(...).json(...)` directly. The backend's default error middleware converts these into properly-formatted HTTP responses with a structured body that the consumer's `ResponseError.fromResponse` parses back into a typed error.

| Class                     | Status              | Use when                                                                             |
| ------------------------- | ------------------- | ------------------------------------------------------------------------------------ |
| `InputError`              | 400                 | Bad request input (malformed key, missing required field, schema validation failure) |
| `AuthenticationError`     | 401                 | Caller's credentials are missing or invalid                                          |
| `NotAllowedError`         | 403                 | Caller is authenticated but not authorized for this resource                         |
| `NotFoundError`           | 404                 | Resource doesn't exist                                                               |
| `ConflictError`           | 409                 | Resource already exists / version mismatch                                           |
| `NotModifiedError`        | 304                 | If-Modified-Since hit                                                                |
| `NotImplementedError`     | 501                 | Endpoint or branch not implemented                                                   |
| `ServiceUnavailableError` | 503                 | Upstream/downstream temporarily unavailable                                          |
| `ForwardedError`          | wraps another error | Re-throwing with added context                                                       |

Examples:

```ts
import {
  InputError,
  NotFoundError,
  NotAllowedError,
  ConflictError,
  ForwardedError,
} from '@backstage/errors';

router.get('/items/:id', async (req, res) => {
  if (!/^[a-z0-9-]+$/.test(req.params.id)) {
    throw new InputError(`invalid id: ${req.params.id}`);
  }

  const item = await loadItem(req.params.id);
  if (!item) {
    throw new NotFoundError(`item ${req.params.id} not found`);
  }

  res.json(item);
});

router.post('/items/:id/publish', async (req, res) => {
  try {
    await externalSystem.publish(req.params.id);
  } catch (e) {
    // Wrap upstream failures with context, preserving the original cause:
    throw new ForwardedError(`failed to publish item ${req.params.id}`, e);
  }
  res.status(202).end();
});
```

Don't:

- `res.status(404).json({ error: '...' })` — bypasses the error middleware and produces a body the consumer's `ResponseError.fromResponse` can't parse.
- `throw new Error('not found')` — ends up as a 500 with no status hint to the consumer.
- `try { ... } catch (e) { res.status(500).json(...) }` — the default error middleware does this for you, with structured logging and a parseable body.

## Post-scaffold usage

For `backend-plugin` template: scaffold an `httpRouter` dep and a hello-world `/health` route. Include `httpRouter.addAuthPolicy({ path: '/health', allow: 'unauthenticated' })` only if the user explicitly asks for an unauthenticated endpoint.

For `backend-plugin-module`, the module typically doesn't mount routes (modules extend an existing plugin via extension points). Skip httpRouter unless the user's intent specifically describes new HTTP routes.

## Tests

Use `startTestBackend` + `supertest`:

```ts
import { startTestBackend, mockServices } from '@backstage/backend-test-utils';
import request from 'supertest';
import { myPlugin } from './plugin';

it('returns ok from /health', async () => {
  const { server } = await startTestBackend({
    features: [myPlugin, mockServices.rootConfig.factory({ data: {} })],
  });

  const response = await request(server).get('/api/my-plugin/health');
  expect(response.status).toBe(200);
  expect(response.body).toEqual({ ok: true });
});

it('rejects unauthenticated POST /items', async () => {
  const { server } = await startTestBackend({ features: [myPlugin] });
  const response = await request(server)
    .post('/api/my-plugin/items')
    .send({ name: 'x' });
  expect(response.status).toBe(401);
});
```

Notes:

- `startTestBackend` provides an `httpRouter` automatically — you don't need to mock it.
- For routes that read auth credentials, install `mockServices.httpAuth.factory(...)` and `mockServices.userInfo.factory()` (see `references/auth.md`). `mockCredentials.user()` and `mockCredentials.service()` produce credential objects you pass on requests.
