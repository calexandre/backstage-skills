# Pattern: Scheduler and lifecycle

Two related services for time-based work. `coreServices.scheduler` runs durable, replica-aware periodic tasks; `coreServices.lifecycle` registers startup/shutdown hooks within a plugin.

## Standalone usage

### Scheduler (`coreServices.scheduler`)

```ts
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';

createBackendPlugin({
  pluginId: 'my-plugin',
  register(env) {
    env.registerInit({
      deps: {
        logger: coreServices.logger,
        scheduler: coreServices.scheduler,
      },
      async init({ logger, scheduler }) {
        await scheduler.scheduleTask({
          id: 'my-plugin-refresh',
          frequency: { minutes: 30 },
          timeout: { minutes: 5 },
          initialDelay: { seconds: 30 },
          fn: async () => {
            logger.info('running scheduled refresh');
            // do work
          },
        });
      },
    });
  },
});
```

Task fields:

- `id` — unique per plugin. Used for cross-replica leader election so only one replica runs the task at a time.
- `frequency` — how often to run. Object form: `{ minutes, seconds, hours, days }` (HumanDuration). Cron form: `{ cron: '0 * * * *' }`.
- `timeout` — max time the task may run before being considered hung.
- `initialDelay` — wait this long after startup before the first run.
- `fn` — the work to do.
- `scope` (optional) — `'global'` (default) for one-runner-across-replicas, `'local'` for one-runner-per-replica.

### Reading schedule from config

For modules whose schedule is configurable, use `readSchedulerServiceTaskScheduleDefinitionFromConfig`:

```ts
import {
  coreServices,
  createBackendModule,
  readSchedulerServiceTaskScheduleDefinitionFromConfig,
} from '@backstage/backend-plugin-api';

const schedule = config.has('catalog.providers.foo.schedule')
  ? readSchedulerServiceTaskScheduleDefinitionFromConfig(
      config.getConfig('catalog.providers.foo.schedule'),
    )
  : { frequency: { minutes: 30 }, timeout: { minutes: 10 } };

await scheduler.scheduleTask({ id: 'foo-refresh', ...schedule, fn: ... });
```

### Lifecycle (`coreServices.lifecycle`)

For startup/shutdown hooks scoped to a single plugin (e.g. opening/closing a connection pool):

```ts
env.registerInit({
  deps: { lifecycle: coreServices.lifecycle, logger: coreServices.logger },
  async init({ lifecycle, logger }) {
    const connection = await openConnection();

    lifecycle.addStartupHook(async () => {
      logger.info('plugin fully started');
    });

    lifecycle.addShutdownHook(async () => {
      logger.info('plugin shutting down');
      await connection.close();
    });
  },
});
```

`coreServices.rootLifecycle` is the same pattern but root-scoped — for hooks that need to fire at the whole-backend level (rare; prefer plugin-scoped `lifecycle`).

## Post-scaffold usage

Invoke this reference when the user's intent describes:

- Periodic work, polling, refreshing, syncing → `scheduler`
- Cleanup or graceful shutdown of resources (connections, watchers, child processes) → `lifecycle`

For a plain `backend-plugin` with no scheduled work, skip both.

## Tests

```ts
import { mockServices } from '@backstage/backend-test-utils';

const scheduler = mockServices.scheduler();

await scheduler.scheduleTask({
  id: 'test-task',
  frequency: { seconds: 1 },
  timeout: { seconds: 5 },
  fn: jest.fn(),
});

// In tests, mockServices.scheduler runs tasks immediately or on demand
// depending on the implementation — check ../backstage/packages/backend-test-utils
// docs for the trigger API.
```

For testing the function inside a scheduled task, prefer **extracting the function and unit-testing it independently** — don't try to test the scheduling itself. The scheduler implementation is a Backstage concern, not your plugin's.

```ts
// Extract:
async function refreshFn(deps: { logger: LoggerService }) {
  // ... actual work
}

// In init:
await scheduler.scheduleTask({
  id: '...',
  frequency,
  timeout,
  fn: () => refreshFn({ logger }),
});

// In tests, just call refreshFn directly with a mock logger.
```
