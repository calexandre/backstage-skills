# Pattern: Plugin definition with `createFrontendPlugin`

The plugin's main entry point is a single `src/plugin.tsx` that calls `createFrontendPlugin` and lists all extensions the plugin contributes. The result is exported as default from `src/index.ts`.

## Standalone usage

### `src/plugin.tsx`

```tsx
import { createFrontendPlugin } from '@backstage/frontend-plugin-api';
import { RiToolsLine } from '@remixicon/react';
import { rootRouteRef, externalDocsRouteRef } from './routes';
import { myPage } from './extensions';
import { myPluginApi } from './apis';

export default createFrontendPlugin({
  pluginId: 'my-plugin',
  title: 'My Plugin',
  icon: <RiToolsLine />,
  info: {
    packageJson: () => import('../package.json'),
  },
  routes: {
    root: rootRouteRef,
  },
  externalRoutes: {
    docs: externalDocsRouteRef,
  },
  extensions: [myPluginApi, myPage],
});
```

Key points:

- `pluginId` is required and is the public identifier other plugins use for route binding (e.g. `my-plugin.root`).
- `title` and `icon` set defaults that propagate to extensions like `PageBlueprint` if those extensions don't override them.
- `routes` map names → route refs the plugin owns (referenced from outside as `<pluginId>.<routeName>`).
- `externalRoutes` map names → external route refs the plugin consumes (apps can override the binding via `bindRoutes`).
- `extensions` is the flat list of every extension this plugin contributes — APIs, pages, sub-pages, nav items, etc.

### `src/index.ts`

```ts
export { default } from './plugin';
export { rootRouteRef } from './routes';
```

### `package.json` exports

If the plugin previously used a separate `./alpha` entry, collapse it into the main entry now that NFS is the only entry point:

```json
{
  "exports": {
    ".": "./src/index.ts",
    "./package.json": "./package.json"
  }
}
```

### Plugin icon

Prefer `@remixicon/react` icons for the plugin icon. If the plugin already had an MUI icon and replacing it is out of scope, you can keep it with `fontSize="inherit"`:

```tsx
import CategoryIcon from '@material-ui/icons/Category';

icon: <CategoryIcon fontSize="inherit" />,
```

Note: this works for the plugin icon and for `PageBlueprint`/`PluginHeader`, but `NavItemBlueprint`'s `icon` prop requires an MUI `IconComponent` type — Remix icons are not compatible there.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds `frontend-plugin` or `frontend-plugin-module`:

1. Create or replace `plugins/<id>/src/plugin.tsx` with the snippet above. Substitute:
   - `pluginId` = the scaffolded plugin's id
   - `title` = a human-readable variant of the id (e.g. `my-plugin` → `My Plugin`)
   - `icon` = `<RiToolsLine />` as a default placeholder (advise the user to pick a more specific icon later)
2. Create `plugins/<id>/src/index.ts` exporting `default` from `./plugin` and any route refs from `./routes`.
3. Update `plugins/<id>/package.json`:
   - Add `@backstage/frontend-plugin-api` to `dependencies`
   - Add `@remixicon/react` to `dependencies`
   - Remove `@backstage/core-plugin-api` if the scaffolder added it (it's not needed for an NFS-only plugin)
   - Update `exports` to point only to `./src/index.ts`
4. Run `yarn install` from the repo root.

## Tests

The plugin definition itself isn't typically unit-tested. Confirm correctness by:

1. Running `yarn tsc` from the repo root — the types catch most structural mistakes (missing `pluginId`, wrong route ref shape, etc.).
2. Loading the plugin in an NFS-style app (`packages/app` if using Spotify Portal Studio or the Backstage demo app) and verifying its routes are discoverable and its `pluginId` matches expectations.
