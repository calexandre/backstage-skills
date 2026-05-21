# Pattern: Backend modules with `createBackendModule`

A backend module extends an existing plugin by registering implementations against that plugin's extension points. The host backend installs the module alongside the target plugin, and the module's deps are resolved from the _target_ plugin's scope (so `coreServices.logger` in a catalog module is the catalog logger).

## Standalone usage

### When a module is the right answer

Use a module (not a full plugin) when:

- You're adding behavior to an existing plugin (a new catalog provider, scaffolder action, auth provider, etc.)
- You want your code to ship as a small package that a backend installs alongside the target plugin
- The target plugin exposes an extension point you can plug into

Use a full plugin (not a module) when:

- You own the route prefix (your code mounts new HTTP routes under `/api/<your-id>/`)
- You define APIs other modules will extend

### `src/module.ts`

Modeled on `plugins/catalog-backend-module-azure-devops-entity-provider/src/module.ts` in this repo:

```ts
import {
  coreServices,
  createBackendModule,
  readSchedulerServiceTaskScheduleDefinitionFromConfig,
} from '@backstage/backend-plugin-api';
import { catalogProcessingExtensionPoint } from '@backstage/plugin-catalog-node';
import { MyEntityProvider } from './providers/MyEntityProvider';

export const catalogModuleMyProvider = createBackendModule({
  pluginId: 'catalog',
  moduleId: 'my-provider',
  register(env) {
    env.registerInit({
      deps: {
        catalog: catalogProcessingExtensionPoint,
        config: coreServices.rootConfig,
        logger: coreServices.logger,
        scheduler: coreServices.scheduler,
      },
      async init({ catalog, config, logger, scheduler }) {
        const schedule = config.has('catalog.providers.myProvider.schedule')
          ? readSchedulerServiceTaskScheduleDefinitionFromConfig(
              config.getConfig('catalog.providers.myProvider.schedule'),
            )
          : { frequency: { minutes: 30 }, timeout: { minutes: 10 } };

        const provider = new MyEntityProvider({ logger, config });
        catalog.addEntityProvider(provider);

        await scheduler.scheduleTask({
          id: 'my-provider-refresh',
          ...schedule,
          fn: () => provider.refresh(logger),
        });
      },
    });
  },
});
```

Key points:

- `pluginId` is the **target** plugin (e.g. `'catalog'`, `'scaffolder'`, `'auth'`), not your own.
- `moduleId` is your module's identifier; must be unique within the target plugin and match `[a-z][a-z0-9-]*`.
- `deps` mixes core services and extension points freely. Extension points come from the target plugin's library package (here `@backstage/plugin-catalog-node`).
- `init` is where you call the extension point's methods (`catalog.addEntityProvider(...)`, `scaffolder.addActions(...)`, etc.).

### `src/index.ts`

```ts
export { catalogModuleMyProvider as default } from './module';
```

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds:

- **`backend-plugin-module`**: emit `src/module.ts` using `createBackendModule`. The target `pluginId` is the value the user gave (e.g. `'catalog'` or `'scaffolder'` or `'auth'`); the `moduleId` is derived from the scaffolded plugin id.
- **`catalog-provider-module`**: target `pluginId` is always `'catalog'`. The module body imports `catalogProcessingExtensionPoint` from `@backstage/plugin-catalog-node` and depends on `catalog`, `logger`, `scheduler`, and `config` by default. Defer to `creating-backstage-catalog-provider` for the full provider class + module wiring.
- **`scaffolder-backend-module`**: target `pluginId` is always `'scaffolder'`. The module imports `scaffolderActionsExtensionPoint` and depends on `scaffolder` and `logger`. Defer to `creating-backstage-scaffolder-action` for the full action + module wiring.

In all cases, ensure `src/index.ts` re-exports the module as default and `package.json` lists `@backstage/backend-plugin-api` plus the target plugin's library package (`@backstage/plugin-catalog-node`, `@backstage/plugin-scaffolder-node`, etc.) in `dependencies`.

## Tests

Use `startTestBackend` with the module installed alongside (a real or mocked) target plugin:

```ts
import { startTestBackend, mockServices } from '@backstage/backend-test-utils';
import { catalogProcessingExtensionPoint } from '@backstage/plugin-catalog-node';
import { catalogModuleMyProvider } from './module';

it('registers the entity provider', async () => {
  const addEntityProvider = jest.fn();

  await startTestBackend({
    features: [
      catalogModuleMyProvider,
      mockServices.rootConfig.factory({
        data: { catalog: { providers: { myProvider: {} } } },
      }),
      // Provide a stub for the extension point:
      {
        $$type: '@backstage/BackendFeature' as const,
        version: 'v1',
        featureType: 'registrations',
        getRegistrations: () => [
          {
            type: 'plugin-v1.1',
            pluginId: 'catalog',
            extensionPoints: [
              {
                extensionPoint: catalogProcessingExtensionPoint,
                factory: () => ({ addEntityProvider, addProcessor: jest.fn() }),
              },
            ],
            init: { deps: {}, func: async () => {} },
          },
        ],
      } as any,
    ],
  });

  expect(addEntityProvider).toHaveBeenCalled();
});
```

For most modules, prefer **unit-testing the underlying class** (`MyEntityProvider`, `MyAction`) directly rather than testing the module wiring — the wiring is mostly declarative and breaks loudly via type errors if mis-spelled.
