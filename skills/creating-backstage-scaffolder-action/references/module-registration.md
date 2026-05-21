# Pattern: Module registration

The module registers the action(s) against the scaffolder plugin's extension point.

## Standalone usage

### `src/module.ts`

```ts
import {
  coreServices,
  createBackendModule,
} from '@backstage/backend-plugin-api';
import { scaffolderActionsExtensionPoint } from '@backstage/plugin-scaffolder-node';
import { myAction } from './actions/myAction';
import { writeReadme } from './actions/writeReadme';

export const scaffolderModuleAcme = createBackendModule({
  pluginId: 'scaffolder',
  moduleId: 'acme',
  register(env) {
    env.registerInit({
      deps: {
        scaffolder: scaffolderActionsExtensionPoint,
        logger: coreServices.logger,
      },
      async init({ scaffolder, logger }) {
        scaffolder.addActions(myAction, writeReadme);
        logger.info('registered acme scaffolder actions');
      },
    });
  },
});
```

Key points:

- `pluginId: 'scaffolder'` is fixed — this module always extends the scaffolder plugin.
- `moduleId` follows `[a-z][a-z0-9-]*`. For a single-namespace module of actions, use the namespace itself (e.g. `'acme'`); for module variants per integration use `'<integration-name>'`.
- `scaffolder.addActions(...)` accepts one or more action objects. Pass them by import — actions are plain objects.
- `scaffolderActionsExtensionPoint` comes from `@backstage/plugin-scaffolder-node` (the library package), not `@backstage/plugin-scaffolder-backend`.

### `src/index.ts`

```ts
export { scaffolderModuleAcme as default } from './module';
```

### Actions that need core services

If an action needs `coreServices.auth`, `coreServices.urlReader`, or any other service, **the action can't depend on services directly** — only the module can. Two options:

**Option A: Inject services into a factory.** Convert the action into a factory function that takes services and returns the action:

```ts
// src/actions/myAction.ts
import { AuthService, DiscoveryService } from '@backstage/backend-plugin-api';
import { createTemplateAction } from '@backstage/plugin-scaffolder-node';

export const createMyAction = (deps: {
  auth: AuthService;
  discovery: DiscoveryService;
}) =>
  createTemplateAction({
    id: 'acme:internal:do-thing',
    schema: { input: { x: z => z.string() } },
    async handler(ctx) {
      const baseUrl = await deps.discovery.getBaseUrl('catalog');
      // ... use auth, discovery, etc.
    },
  });
```

Then in the module:

```ts
deps: {
  scaffolder: scaffolderActionsExtensionPoint,
  auth: coreServices.auth,
  discovery: coreServices.discovery,
},
async init({ scaffolder, auth, discovery }) {
  scaffolder.addActions(createMyAction({ auth, discovery }));
},
```

**Option B: Use closures inside the module's init.** Inline the `createTemplateAction` call inside `init` so it captures services from the closure. Fine for simple cases; awkward to unit-test.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `scaffolder-backend-module`:

1. Create `plugins/<scaffolded-id>/src/module.ts` using the template above.
2. `pluginId` is always `'scaffolder'`. `moduleId` is derived from the scaffolded plugin id (drop the `scaffolder-backend-module-` prefix).
3. `addActions(...)` lists every action the module exports.
4. Ensure `src/index.ts` re-exports the module as default.
5. Add `@backstage/backend-plugin-api` and `@backstage/plugin-scaffolder-node` to `dependencies`.

## Tests

```ts
import { mockServices, startTestBackend } from '@backstage/backend-test-utils';
import { scaffolderActionsExtensionPoint } from '@backstage/plugin-scaffolder-node';
import { scaffolderModuleAcme } from './module';

it('registers all actions', async () => {
  const addActions = jest.fn();

  await startTestBackend({
    features: [
      scaffolderModuleAcme,
      mockServices.rootConfig.factory({ data: {} }),
      {
        $$type: '@backstage/BackendFeature' as const,
        version: 'v1',
        featureType: 'registrations',
        getRegistrations: () => [
          {
            type: 'plugin-v1.1',
            pluginId: 'scaffolder',
            extensionPoints: [
              {
                extensionPoint: scaffolderActionsExtensionPoint,
                factory: () => ({ addActions }),
              },
            ],
            init: { deps: {}, func: async () => {} },
          },
        ],
      } as any,
    ],
  });

  expect(addActions).toHaveBeenCalled();
});
```

For per-action behavior testing, prefer unit tests against the action's `handler` directly (see `references/template-action.md`) — module wiring is mostly declarative.
