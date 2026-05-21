# Pattern: Logger and config

The two services every plugin uses. Both are dependable from any `register` block.

## Standalone usage

### Logger (`coreServices.logger`)

```ts
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';

createBackendPlugin({
  pluginId: 'my-plugin',
  register(env) {
    env.registerInit({
      deps: { logger: coreServices.logger },
      async init({ logger }) {
        logger.info('starting up');
        logger.warn('something to watch', { extra: 'context' });
        logger.error(new Error('boom'));

        const childLogger = logger.child({ subsystem: 'reconciler' });
        childLogger.info('subsystem-tagged log');
      },
    });
  },
});
```

Key points:

- The plugin-scoped logger is pre-tagged with `pluginId` — no need to add it manually.
- Methods: `info`, `warn`, `error`, `debug`. All accept `(msg, meta?)` or `(error)`.
- `logger.child({ ... })` returns a logger with additional persistent metadata.

### Config (`coreServices.rootConfig`)

```ts
env.registerInit({
  deps: { config: coreServices.rootConfig },
  async init({ config }) {
    const apiUrl = config.getString('myPlugin.apiUrl');
    const timeout = config.getOptionalNumber('myPlugin.timeoutMs') ?? 5000;
    const flag = config.getOptionalBoolean('myPlugin.featureX') ?? false;

    if (config.has('myPlugin.subsection')) {
      const sub = config.getConfig('myPlugin.subsection');
      const items = sub.getStringArray('items');
    }
  },
});
```

Key points:

- `rootConfig` is root-scoped (single shared instance), but you depend on it like any other service.
- Methods: `getString`, `getOptionalString`, `getNumber`, `getOptionalNumber`, `getBoolean`, `getOptionalBoolean`, `getStringArray`, `getOptionalStringArray`, `getConfig` (returns a sub-Config), `getOptionalConfig`, `has`.
- Reads from `app-config.yaml` resolved at backend startup. To opt into runtime updates, the implementation supports observation but plugins typically read once during init.

### Declaring the config schema (`config.d.ts`)

Every plugin that reads from `rootConfig` should publish a typed schema so adopters get validation and IDE autocomplete in their `app-config.yaml`. Place a `config.d.ts` file at the package root (alongside `package.json`, NOT in `src/`):

```ts
// config.d.ts
import { HumanDuration } from '@backstage/types';

export interface Config {
  myPlugin?: {
    /**
     * Base URL of the upstream service this plugin reads from.
     */
    apiUrl: string;
    /**
     * Optional request timeout. Accepts e.g. `{ seconds: 30 }`.
     * @default { seconds: 5 }
     */
    timeout?: HumanDuration;
    /**
     * Feature flag. When true, additional behavior X is enabled.
     * @default false
     */
    featureX?: boolean;
  };
}
```

Then wire the file into `package.json`:

```json
{
  "files": ["dist", "config.d.ts"],
  "configSchema": "config.d.ts"
}
```

What this does:

- `configSchema` tells Backstage's config validation tooling which file to read; types from the `Config` interface are merged with all other plugins' schemas to produce the full app-config type.
- Including `config.d.ts` in `files` makes it ship with the published package so adopters get the schema when they install the plugin.
- The interface is named `Config` (not `MyPluginConfig`) and uses TypeScript's declaration merging — multiple plugins all extend the same global `Config` interface, each contributing their top-level key.
- Use JSDoc comments — they surface as descriptions in the schema and as hover hints in editors.
- Mark plugin-level keys as optional (`myPlugin?:`) so adopters who haven't installed the plugin don't fail validation.

## Post-scaffold usage

For `backend-plugin` and `backend-plugin-module` templates, include `logger` as a default dep, and `config` if the plugin reads any configuration:

```ts
deps: {
  logger: coreServices.logger,
  config: coreServices.rootConfig,
  // ... other deps based on intent
},
```

The scaffolded `init` body should `logger.info('starting <id>')` and read at least one config key if `config` is included.

When the plugin reads config, also create `<plugin-package>/config.d.ts` declaring the typed schema, and update `package.json` to list `config.d.ts` in `files` and set `"configSchema": "config.d.ts"`. Use the scaffolded plugin id as the top-level key in the `Config` interface (e.g. `myPlugin?: { ... }` for plugin id `my-plugin`).

## Tests

For unit tests of helper functions that take a logger or config:

```ts
import { mockServices } from '@backstage/backend-test-utils';

const logger = mockServices.rootLogger().child({ pluginId: 'my-plugin' });
const config = mockServices.rootConfig({
  data: { myPlugin: { apiUrl: 'http://x' } },
});

// Pass to your function under test:
await myHelper({ logger, config });
expect(config.getString('myPlugin.apiUrl')).toBe('http://x');
```

For integration tests with `startTestBackend`:

```ts
import { startTestBackend, mockServices } from '@backstage/backend-test-utils';
import { myPlugin } from './plugin';

await startTestBackend({
  features: [
    myPlugin,
    mockServices.rootConfig.factory({
      data: { myPlugin: { apiUrl: 'http://x' } },
    }),
  ],
});
```

Notes:

- `mockServices.rootLogger()` returns a Jest-mock-friendly RootLoggerService. There is no `mockServices.logger()` — for plugin-scoped logger, the test backend derives it from `rootLogger` automatically.
- `mockServices.rootConfig({ data })` returns the service for direct injection into helpers; `mockServices.rootConfig.factory({ data })` returns a factory you pass to `startTestBackend`'s `features` array.
