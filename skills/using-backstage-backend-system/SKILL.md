---
name: using-backstage-backend-system
description: Use when defining or authoring a Backstage backend plugin or module. Covers createBackendPlugin and createBackendModule from @backstage/backend-plugin-api, service registration via coreServices.* refs, and exposing extension points with createExtensionPoint. Triggers on imports from @backstage/backend-plugin-api or when writing src/plugin.ts, src/module.ts, src/index.ts in a backend package.
---

# Backstage Backend System Patterns

## When to use this skill

- Defining a new backend plugin (`src/plugin.ts`)
- Defining a backend module (`src/module.ts`) that extends another plugin
- Choosing between a full plugin vs a module
- Exposing extension points so other modules can extend your plugin

## Choose the right pattern

| Building...                                          | Pattern                                                 | Reference                       |
| ---------------------------------------------------- | ------------------------------------------------------- | ------------------------------- |
| The plugin itself                                    | `createBackendPlugin({ pluginId, register })`           | references/plugin-definition.md |
| A module that extends another plugin                 | `createBackendModule({ pluginId, moduleId, register })` | references/modules.md           |
| An extension point so other modules can extend yours | `createExtensionPoint<T>({ id })`                       | references/extension-points.md  |

## Two invocation modes

- **Standalone**: editing an existing backend plugin/module.
- **Post-scaffold** (called by `creating-backstage-plugin` for `backend-plugin`, `backend-plugin-module`, `catalog-provider-module`, `scaffolder-backend-module`): wire up the canonical structure (`src/plugin.ts` or `src/module.ts`, `src/index.ts` re-export).

## Common gotchas

- All wiring imports come from `@backstage/backend-plugin-api`, not `@backstage/core-plugin-api` (which is frontend-only).
- The `register(env)` callback receives an environment object with `env.registerInit({ deps, async init })` and `env.registerExtensionPoint(point, impl)`. `deps` is a map of service refs the init function needs; `init` receives the resolved instances.
- `registerInit` may be called only once per plugin/module. `registerExtensionPoint` must be called before `registerInit`.
- Plugin-scoped vs root-scoped services: most refs (`logger`, `httpRouter`, `database`, `scheduler`, `auth`, `httpAuth`, `userInfo`, `cache`, `discovery`, `urlReader`, `lifecycle`, `permissions`) are plugin-scoped — each plugin gets its own instance with plugin-aware behavior (e.g. logger pre-tagged with `pluginId`, database giving a per-plugin schema). Six refs (`rootConfig`, `rootLogger`, `rootLifecycle`, `rootHealth`, `rootHttpRouter`, `rootInstanceMetadata`) are root-scoped — single shared instance. Both can be depended on from a plugin's `register` block; the distinction matters for behavior, not for injection syntax.
- A plugin's `pluginId` and a module's `moduleId` must match the pattern `[a-z][a-z0-9-]*` (letters, digits, dashes; starts with a letter).
- Modules depend on the extension points exported by the target plugin's library package (e.g. `@backstage/plugin-catalog-node`), not on the plugin's main package.
