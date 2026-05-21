# Pattern: Backend plugin definition with `createBackendPlugin`

A backend plugin is defined by a single `createBackendPlugin` call in `src/plugin.ts`, exported as default from `src/index.ts`. The plugin declares its dependencies on services from `coreServices`, and its `init` function receives the resolved instances.

## Standalone usage

### `src/plugin.ts`

```ts
import {
  createBackendPlugin,
  coreServices,
} from '@backstage/backend-plugin-api';
import { Router } from 'express';

export const myPlugin = createBackendPlugin({
  pluginId: 'my-plugin',
  register(env) {
    env.registerInit({
      deps: {
        logger: coreServices.logger,
        httpRouter: coreServices.httpRouter,
        config: coreServices.rootConfig,
      },
      async init({ logger, httpRouter, config }) {
        logger.info('starting my-plugin');

        const router = Router();
        router.get('/health', (_, res) => res.json({ ok: true }));
        httpRouter.use(router);
      },
    });
  },
});
```

Key points:

- `pluginId` is required and is the public identifier other plugins/modules use (e.g. `my-plugin.<routeName>` for cross-plugin links). It must match `[a-z][a-z0-9-]*`.
- `register(env)` is called once when the backend builds the plugin. Inside it, call `env.registerInit({ deps, init })` exactly once.
- `deps` is a map: each key is a name you'll receive in `init`; each value is a service ref from `coreServices` (or an extension point from another plugin's library).
- `init` is async. It runs at backend startup and is where you mount routes, schedule tasks, register extension point implementations, etc.

### `src/index.ts`

```ts
export { myPlugin as default } from './plugin';
```

Exporting as default makes `import myPlugin from '@your/plugin-package'` work, which is how the host backend installs it via `backend.add(myPlugin)`.

### `package.json`

The plugin's `package.json` should:

- Have `"backstage": { "role": "backend-plugin" }` (set by `yarn new`)
- Depend on `@backstage/backend-plugin-api`
- Depend on `@backstage/backend-defaults` only for tests / dev — implementations are wired by the host backend, not the plugin

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `backend-plugin`:

1. Locate `plugins/<id>-backend/src/plugin.ts` (the scaffolded entry point).
2. Replace its body with the snippet above. Use:
   - `pluginId` = the scaffolded plugin id (without `-backend` suffix)
   - Default deps: `logger` and `httpRouter`. Add `config: coreServices.rootConfig` if intent involves config-driven behavior.
   - In `init`, mount a sample `/health` router.
3. Confirm `plugins/<id>-backend/src/index.ts` re-exports the plugin as default.
4. Confirm `@backstage/backend-plugin-api` is in `dependencies`. If catalog/auth/database deps are added, those packages aren't required — only the refs come from `@backstage/backend-plugin-api`.

## Tests

Use `startTestBackend` from `@backstage/backend-test-utils` to spin up an integration test backend with the plugin and any service mocks needed:

```ts
import { startTestBackend, mockServices } from '@backstage/backend-test-utils';
import request from 'supertest';
import { myPlugin } from './plugin';

it('exposes /health', async () => {
  const { server } = await startTestBackend({
    features: [myPlugin, mockServices.rootConfig.factory({ data: {} })],
  });

  const response = await request(server).get('/api/my-plugin/health');
  expect(response.status).toBe(200);
  expect(response.body).toEqual({ ok: true });
});
```

Notes:

- `startTestBackend` returns `{ server, ... }`; `server` is the underlying HTTP server you can hit with `supertest`.
- Plugin routes are mounted under `/api/<pluginId>/` by `coreServices.httpRouter`.
- `mockServices.*` factory variants exist for most core services. If a service isn't explicitly mocked, the test backend uses a sensible default (e.g. an in-memory cache).
