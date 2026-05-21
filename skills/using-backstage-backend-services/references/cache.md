# Pattern: Cache

`coreServices.cache` is a plugin-scoped key-value cache. Each plugin gets its own keyspace, so keys can't collide across plugins. The implementation defaults to in-memory but can be backed by Redis or Memcached via app-config.

## Standalone usage

```ts
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';

createBackendPlugin({
  pluginId: 'my-plugin',
  register(env) {
    env.registerInit({
      deps: { cache: coreServices.cache, logger: coreServices.logger },
      async init({ cache, logger }) {
        const key = 'expensive:query:foo';

        let value = await cache.get<{ result: string }>(key);
        if (!value) {
          logger.info('cache miss; computing');
          value = { result: await computeExpensive() };
          await cache.set(key, value, { ttl: { hours: 1 } });
        }

        await cache.delete(key); // remove from cache
      },
    });
  },
});
```

Methods:

- `cache.get<T>(key)` → `Promise<T | undefined>`
- `cache.set(key, value, options?)` → `Promise<void>` — `options.ttl` accepts a HumanDuration (`{ minutes, seconds, hours, days }`) or a number of milliseconds.
- `cache.delete(key)` → `Promise<void>`

Key points:

- Values are JSON-serialized. Don't store class instances, `Date` objects, or `Map`/`Set` — convert to plain objects first.
- Plugin-scoped: `cache.set('foo', ...)` from plugin A doesn't collide with `cache.set('foo', ...)` from plugin B.
- Default implementation is in-memory and resets on backend restart. Configure Redis/Memcached via `backend.cache.store` in `app-config.yaml` for cross-replica or persistent caching.

## Post-scaffold usage

Only invoke if the user's intent describes caching expensive computations or memoizing external API responses. Otherwise skip.

When applicable:

1. Add `cache: coreServices.cache` to deps.
2. Wrap the expensive operation in a get-or-compute pattern as above.
3. Pick a sensible TTL — defaults are intentionally not provided, so the plugin makes an explicit choice.

## Tests

```ts
import { mockServices } from '@backstage/backend-test-utils';

// mockServices.cache is not in the public mockServices namespace as of
// Backstage 1.50 — startTestBackend provides an in-memory cache automatically.
// For unit tests of helper functions that take a cache, pass a stub:

const cache = {
  get: jest.fn(),
  set: jest.fn(),
  delete: jest.fn(),
} as any;

await myHelper({ cache });
expect(cache.set).toHaveBeenCalledWith('expected-key', expect.any(Object), {
  ttl: { hours: 1 },
});
```

For integration tests with `startTestBackend`, the default in-memory cache is sufficient; no factory installation is needed.
