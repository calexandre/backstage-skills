# Pattern: Sub-pages with `SubPageBlueprint`

Use `SubPageBlueprint` when the plugin has top-level tabs (e.g. Overview / Settings) that users navigate between via the page header. The framework renders the tabs from the parent `PageBlueprint`'s sub-pages.

## Standalone usage

### Decide first

Use `SubPageBlueprint` when:

- The sub-routes represent top-level tabs of the plugin
- Users navigate between them via the header

Keep internal routing within a single `PageBlueprint` `loader` when:

- Routes are detail/drill-down pages (e.g. `/my-plugin/items/:id`)
- The routing is deeply nested or dynamic

### Parent page (no loader, framework renders tabs)

```tsx
// src/extensions.tsx
import {
  PageBlueprint,
  SubPageBlueprint,
} from '@backstage/frontend-plugin-api';
import { rootRouteRef } from './routes';

export const myPluginPage = PageBlueprint.make({
  params: {
    path: '/my-plugin',
    routeRef: rootRouteRef,
    // no loader — sub-pages will be rendered as tabs
  },
});
```

### Sub-pages

```tsx
export const overviewSubPage = SubPageBlueprint.make({
  name: 'overview',
  params: {
    path: 'overview',
    title: 'Overview',
    loader: () =>
      import('./components/OverviewPage').then(m => <m.OverviewPageContent />),
  },
});

export const settingsSubPage = SubPageBlueprint.make({
  name: 'settings',
  params: {
    path: 'settings',
    title: 'Settings',
    loader: () =>
      import('./components/SettingsPage').then(m => <m.SettingsPageContent />),
  },
});
```

How this works:

- `PageBlueprint` **without a loader** automatically renders its sub-pages as tabs
- The first sub-page becomes the default (index redirect)
- Each `SubPageBlueprint` gets a tab in the header, labeled with its `title`
- Sub-page `path` values are **relative** (no leading `/`)
- Sub-page components render **content only** — no `Page`, `Header`, or `HeaderTabs`

If the sub-page content needs padding, wrap with `Container` from `@backstage/ui` inside the component.

### Wiring into the plugin

```tsx
// src/plugin.tsx
import { myPluginPage, overviewSubPage, settingsSubPage } from './extensions';

export default createFrontendPlugin({
  pluginId: 'my-plugin',
  // ...
  extensions: [myPluginPage, overviewSubPage, settingsSubPage],
});
```

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin` AND the user's intent describes top-level tabbed sections:

1. In `plugins/<id>/src/extensions.tsx`, change the parent `PageBlueprint` to omit its `loader` param.
2. Add a `SubPageBlueprint` for each tab the user described (defaults: at least `overview` if intent is unspecific).
3. Create one component file per sub-page under `plugins/<id>/src/components/<TabName>Page/<TabName>Page.tsx`. Each component renders content only.
4. Add all sub-page extensions to `src/plugin.tsx`'s `extensions` array.

If the user did not request tabs, skip `SubPageBlueprint` entirely and use the simple `PageBlueprint` pattern from `references/page-extensions.md`.

## Tests

Sub-page components are plain React components — test them in isolation with `renderInTestApp`:

```tsx
import { renderInTestApp } from '@backstage/frontend-test-utils';
import { screen } from '@testing-library/react';
import { OverviewPageContent } from './OverviewPage';

it('renders the overview content', async () => {
  renderInTestApp(<OverviewPageContent />);
  expect(await screen.findByText(/overview/i)).toBeInTheDocument();
});
```

Tab navigation between sub-pages is integration-level — covered by an end-to-end test inside a host app, not a unit test.
