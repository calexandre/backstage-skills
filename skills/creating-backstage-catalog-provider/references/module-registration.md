# Pattern: Module registration

The module wires the provider into the catalog backend. Wiring differs by provider model (see `references/entity-provider.md` for the model choice).

## Normal provider — `src/module.ts`

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
        const providerConfig = config.getOptionalConfig(
          'catalog.providers.myProvider',
        );

        const schedule = providerConfig?.has('schedule')
          ? readSchedulerServiceTaskScheduleDefinitionFromConfig(
              providerConfig.getConfig('schedule'),
            )
          : { frequency: { minutes: 30 }, timeout: { minutes: 10 } };

        const apiUrl =
          providerConfig?.getString('apiUrl') ??
          (() => {
            throw new Error('catalog.providers.myProvider.apiUrl is required');
          })();

        catalog.addEntityProvider(
          new MyEntityProvider({
            config: { apiUrl },
            logger,
            schedule: scheduler.createScheduledTaskRunner(schedule),
          }),
        );
      },
    });
  },
});
```

Key points:

- `pluginId: 'catalog'` is fixed — this module always extends the catalog plugin.
- `moduleId` follows the format `<source-system>` or `<source-system>-entity-provider` for clarity. Must match `[a-z][a-z0-9-]*`.
- `catalogProcessingExtensionPoint` exposes `addEntityProvider`, `addProcessor`, and other methods. Verify the exact set in `../backstage/plugins/catalog-node/src/extensions.ts`.
- `scheduler.createScheduledTaskRunner(schedule)` returns a `SchedulerServiceTaskRunner` — pass it to the provider's constructor. The provider's `connect()` calls `this.schedule.run({ id, fn })` to register the work.
- `readSchedulerServiceTaskScheduleDefinitionFromConfig` parses a `schedule` block from app-config (`{ frequency, timeout, initialDelay, scope }`). Use it so adopters can override the schedule without editing code.

### `src/index.ts`

```ts
export { catalogModuleMyProvider as default } from './module';
```

### Adopter-side configuration

The adopter installs the module in `packages/backend/src/index.ts` and adds config in `app-config.yaml`:

```yaml
# app-config.yaml
catalog:
  providers:
    myProvider:
      apiUrl: https://api.example.com/services
      schedule:
        frequency: { minutes: 30 }
        timeout: { minutes: 10 }
```

## Incremental provider — `src/module.ts`

For incremental providers, register against `incrementalIngestionProvidersExtensionPoint` instead of `catalogProcessingExtensionPoint`. The framework (from `@backstage/plugin-catalog-backend-module-incremental-ingestion`) wraps your provider with its own scheduler and database-backed cursor persistence — you don't pass a `SchedulerServiceTaskRunner`.

```ts
import {
  coreServices,
  createBackendModule,
} from '@backstage/backend-plugin-api';
import { incrementalIngestionProvidersExtensionPoint } from '@backstage/plugin-catalog-backend-module-incremental-ingestion';
import { MyIncrementalProvider } from './providers/MyIncrementalProvider';

export const catalogModuleMyIncrementalProvider = createBackendModule({
  pluginId: 'catalog',
  moduleId: 'my-incremental-provider',
  register(env) {
    env.registerInit({
      deps: {
        incremental: incrementalIngestionProvidersExtensionPoint,
        config: coreServices.rootConfig,
        logger: coreServices.logger,
      },
      async init({ incremental, config, logger }) {
        const providerConfig = config.getConfig(
          'catalog.providers.myIncrementalProvider',
        );

        const apiUrl = providerConfig.getString('apiUrl');
        const apiToken = providerConfig.getString('apiToken');
        const pageSize = providerConfig.getOptionalNumber('pageSize') ?? 100;

        incremental.addProvider({
          provider: new MyIncrementalProvider(
            { apiUrl, apiToken, pageSize },
            logger,
          ),
          options: {
            // Each burst runs `next` repeatedly for up to this long...
            burstLength: { seconds: 30 },
            // ...with at least this much wait between bursts.
            burstInterval: { seconds: 5 },
            // After completing a full ingestion (last `next` returned done:true),
            // wait this long before starting again.
            restLength: { hours: 1 },
            // Optional: retry backoff steps after errors.
            backoff: [
              { minutes: 1 },
              { minutes: 5 },
              { minutes: 30 },
              { hours: 3 },
            ],
            // Safety: refuse to remove entities if doing so would delete more
            // than this percent of what the provider currently owns.
            rejectRemovalsAbovePercentage: 50,
          },
        });
      },
    });
  },
});
```

Key differences from the normal pattern:

- Depend on `incrementalIngestionProvidersExtensionPoint` (from `@backstage/plugin-catalog-backend-module-incremental-ingestion`), not `catalogProcessingExtensionPoint`.
- No `scheduler` dep, no `SchedulerServiceTaskRunner`. The framework runs the loop.
- `addProvider({ provider, options })` takes both the provider instance AND framework-controlled options (`burstLength`, `burstInterval`, `restLength`, `backoff`, etc.).
- Adopters MUST install `catalogModuleIncrementalIngestionEntityProvider` (the wrapper module from the same package) into their backend in addition to your module — document this in your README.

### Adopter-side configuration (incremental)

```yaml
# app-config.yaml
catalog:
  providers:
    myIncrementalProvider:
      apiUrl: https://api.example.com
      apiToken: ${MY_API_TOKEN}
      pageSize: 200
```

The adopter's `packages/backend/src/index.ts` adds both:

```ts
import { catalogModuleIncrementalIngestionEntityProvider } from '@backstage/plugin-catalog-backend-module-incremental-ingestion';
import { catalogModuleMyIncrementalProvider } from '@your-scope/plugin-catalog-backend-module-my-source';

backend.add(catalogModuleIncrementalIngestionEntityProvider);
backend.add(catalogModuleMyIncrementalProvider);
```

The order doesn't matter — the framework module exposes the extension point your module needs at register time.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `catalog-provider-module`:

1. **Match the module wiring to the provider model chosen in `references/entity-provider.md`.** Normal provider → `catalogProcessingExtensionPoint` + scheduler-driven; incremental provider → `incrementalIngestionProvidersExtensionPoint` + framework-driven.
2. Create `plugins/<scaffolded-id>/src/module.ts` using the matching template above.
3. `pluginId` is always `'catalog'`. `moduleId` is derived from the scaffolded plugin id (drop the `catalog-backend-module-` prefix).
4. The config key under `catalog.providers.<key>` should be a camelCase or kebab-case version of the source system name.
5. Ensure `src/index.ts` re-exports the module as default.
6. Dependencies in `package.json`:
   - **Normal:** `@backstage/backend-plugin-api`, `@backstage/plugin-catalog-node`, `@backstage/catalog-model`.
   - **Incremental:** `@backstage/backend-plugin-api`, `@backstage/plugin-catalog-backend-module-incremental-ingestion`, `@backstage/catalog-model`.
7. **For incremental modules**, write a short paragraph in the plugin's `README.md` telling adopters to also install `catalogModuleIncrementalIngestionEntityProvider`. Without that, `addProvider` calls go nowhere.

## Tests

### Normal module

```ts
import { mockServices, startTestBackend } from '@backstage/backend-test-utils';
import { catalogProcessingExtensionPoint } from '@backstage/plugin-catalog-node';
import { catalogModuleMyProvider } from './module';

it('registers an EntityProvider', async () => {
  const addEntityProvider = jest.fn();
  const addProcessor = jest.fn();

  await startTestBackend({
    features: [
      catalogModuleMyProvider,
      mockServices.rootConfig.factory({
        data: {
          catalog: {
            providers: {
              myProvider: { apiUrl: 'http://x/items' },
            },
          },
        },
      }),
      // Inline a stub plugin that exposes the catalog extension point:
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
                factory: () => ({
                  addEntityProvider,
                  addProcessor,
                }),
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

### Incremental module

For the incremental module, assert that `addProvider` was called with the expected `provider` and `options`:

```ts
import { mockServices, startTestBackend } from '@backstage/backend-test-utils';
import { incrementalIngestionProvidersExtensionPoint } from '@backstage/plugin-catalog-backend-module-incremental-ingestion';
import { catalogModuleMyIncrementalProvider } from './module';

it('registers an IncrementalEntityProvider with the configured options', async () => {
  const addProvider = jest.fn();

  await startTestBackend({
    features: [
      catalogModuleMyIncrementalProvider,
      mockServices.rootConfig.factory({
        data: {
          catalog: {
            providers: {
              myIncrementalProvider: {
                apiUrl: 'http://x',
                apiToken: 't',
              },
            },
          },
        },
      }),
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
                extensionPoint: incrementalIngestionProvidersExtensionPoint,
                factory: () => ({ addProvider }),
              },
            ],
            init: { deps: {}, func: async () => {} },
          },
        ],
      } as any,
    ],
  });

  expect(addProvider).toHaveBeenCalledWith(
    expect.objectContaining({
      provider: expect.objectContaining({
        getProviderName: expect.any(Function),
        next: expect.any(Function),
        around: expect.any(Function),
      }),
      options: expect.objectContaining({
        burstLength: expect.any(Object),
        restLength: expect.any(Object),
      }),
    }),
  );
});
```

For more focused tests, prefer unit-testing the provider class directly (see `references/entity-provider.md`) — module wiring is mostly declarative and tends to fail loudly via type errors, not runtime bugs.
