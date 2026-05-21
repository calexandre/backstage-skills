# Pattern: Extension points

Extension points let your plugin be extended by modules at install time. The plugin defines a typed extension point and registers a default implementation; modules in the same backend depend on the extension point's symbol and call its methods to plug in additional behavior.

## Standalone usage

### When to expose an extension point

- Other modules legitimately need to plug in custom behavior (catalog providers, scaffolder actions, auth providers, etc.).
- The plugin can express the contract as a small interface (a few `addX(...)` methods).
- You're willing to commit to that interface as a public API.

If only your plugin will ever extend itself, skip extension points and just inline the logic.

### Defining the extension point

The extension point lives in the plugin's **library** package (typically `@your/plugin-foo-node` for a backend plugin `@your/plugin-foo-backend`), not the plugin package itself, so modules can depend on it without pulling in the whole plugin.

```ts
// In @your/plugin-foo-node/src/extensions.ts
import { createExtensionPoint } from '@backstage/backend-plugin-api';

export interface FooProvider {
  getName(): string;
  fetch(): Promise<unknown[]>;
}

export interface FooExtensionPoint {
  addProvider(provider: FooProvider): void;
}

export const fooExtensionPoint = createExtensionPoint<FooExtensionPoint>({
  id: 'foo.providers',
});
```

`id` should match the pattern `<pluginId>.<extension-point-name>` to keep names predictable.

### Registering the implementation in the plugin

Inside the plugin's `register(env)` callback, register the extension point's implementation **before** `registerInit`:

```ts
// In @your/plugin-foo-backend/src/plugin.ts
import {
  createBackendPlugin,
  coreServices,
} from '@backstage/backend-plugin-api';
import { fooExtensionPoint, FooProvider } from '@your/plugin-foo-node';

export const fooPlugin = createBackendPlugin({
  pluginId: 'foo',
  register(env) {
    const providers: FooProvider[] = [];

    env.registerExtensionPoint(fooExtensionPoint, {
      addProvider(provider) {
        providers.push(provider);
      },
    });

    env.registerInit({
      deps: { logger: coreServices.logger },
      async init({ logger }) {
        logger.info(`foo plugin started with ${providers.length} providers`);
        // Use `providers` from here on — closure captured.
      },
    });
  },
});
```

Key points:

- `registerExtensionPoint` must be called before `registerInit` (the framework throws otherwise).
- The extension point's implementation is just an object matching the interface. Closure-captured state (here `providers`) is the standard way to collect calls.
- By the time `init` runs, all modules' `init` methods have already populated the extension point.

### Consuming the extension point in a module

Modules import the symbol from the library package and add it as a dep:

```ts
import { createBackendModule, coreServices } from '@backstage/backend-plugin-api';
import { fooExtensionPoint } from '@your/plugin-foo-node';

export const fooModuleMyProvider = createBackendModule({
  pluginId: 'foo',
  moduleId: 'my-provider',
  register(env) {
    env.registerInit({
      deps: { foo: fooExtensionPoint, logger: coreServices.logger },
      async init({ foo, logger }) {
        foo.addProvider({
          getName: () => 'my-provider',
          fetch: async () => [...],
        });
      },
    });
  },
});
```

## Post-scaffold usage

Only relevant for the `backend-plugin` template, and only when the user's intent describes an extensible plugin (e.g. "plugin that other modules can extend with X", "registry of Y"). Otherwise skip — most backend plugins don't need extension points.

When applicable:

1. Create a separate library package or `src/extensions.ts` in the plugin.
2. Define the interface and call `createExtensionPoint<...>({ id: '<pluginId>.<name>' })`.
3. In `src/plugin.ts`, before `registerInit`, call `env.registerExtensionPoint(yourExtensionPoint, impl)`.

## Tests

Extension points themselves are constants — no tests needed. Test the plugin by:

1. **Plugin unit test**: instantiate the plugin in `startTestBackend`, install a small inline module that exercises the extension point, and assert observable behavior.
2. **Module unit test**: test each module independently (see `references/modules.md`).
