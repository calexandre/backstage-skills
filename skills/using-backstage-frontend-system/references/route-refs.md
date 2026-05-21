# Pattern: Route refs

Route refs are the typed identifiers used by `PageBlueprint` to register routes, by other plugins to link to your pages, and by your plugin to link out to other plugins' pages.

## Standalone usage

### `src/routes.ts`

```ts
import {
  createRouteRef,
  createSubRouteRef,
  createExternalRouteRef,
} from '@backstage/frontend-plugin-api';

// Route refs the plugin owns
export const rootRouteRef = createRouteRef();
export const detailsRouteRef = createSubRouteRef({
  path: '/details/:id',
  parent: rootRouteRef,
});

// Route refs to other plugins' pages
export const externalDocsRouteRef = createExternalRouteRef({
  defaultTarget: 'techdocs.docRoot',
});
```

Key points:

- `createRouteRef()` takes no `id`. The id is derived from the extension that consumes it.
- `createSubRouteRef.path` must start with `/` and must NOT end with `/`.
- `createExternalRouteRef()` takes no `id` or `optional` flag.

### `defaultTarget` for external routes

Always set `defaultTarget` to the most common binding so apps don't need explicit `bindRoutes` calls for standard plugin combinations:

```ts
export const createComponentRouteRef = createExternalRouteRef({
  defaultTarget: 'scaffolder.root',
});

export const viewTechDocRouteRef = createExternalRouteRef({
  params: ['namespace', 'kind', 'name'],
  defaultTarget: 'techdocs.docRoot',
});

export const catalogEntityRouteRef = createExternalRouteRef({
  params: ['namespace', 'kind', 'name'],
  defaultTarget: 'catalog.catalogEntity',
});
```

`defaultTarget` uses the format `<pluginId>.<routeName>`, matching a key in the target plugin's `routes` map (see `references/plugin-definition.md`). The default is only activated when the target plugin is installed; otherwise the route stays unbound and `useRouteRef` returns `undefined`.

### Wiring routes into the plugin

Routes are exposed by the plugin definition's `routes` and `externalRoutes` maps:

```tsx
// src/plugin.tsx
export default createFrontendPlugin({
  pluginId: 'my-plugin',
  routes: {
    root: rootRouteRef,
    details: detailsRouteRef,
  },
  externalRoutes: {
    docs: externalDocsRouteRef,
  },
  extensions: [...],
});
```

External plugins reference your routes as `my-plugin.root`, `my-plugin.details`. App-level `bindRoutes` configures cross-plugin bindings for any external route that needs an override.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds `frontend-plugin` or `frontend-plugin-module`:

1. Create `plugins/<id>/src/routes.ts` with at minimum a `rootRouteRef`:

   ```ts
   import { createRouteRef } from '@backstage/frontend-plugin-api';

   export const rootRouteRef = createRouteRef();
   ```

2. If the scaffold generated example sub-routes, also export `createSubRouteRef` entries pointing at `rootRouteRef`.
3. Reference `rootRouteRef` from `src/plugin.tsx`'s `routes.root` (see `references/plugin-definition.md`).

## Tests

Route refs themselves are constants — no tests needed. Routing behavior is tested at the page extension level (see `references/page-extensions.md`).
