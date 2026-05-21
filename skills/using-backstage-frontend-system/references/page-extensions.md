# Pattern: Page extensions with `PageBlueprint`

Pages are extensions that bind a route ref to a component. The framework handles the page chrome (`PluginHeader` and layout) — the page component renders content only.

## Standalone usage

### Simple page

```tsx
// src/extensions.tsx
import { PageBlueprint } from '@backstage/frontend-plugin-api';
import { rootRouteRef } from './routes';

export const myPage = PageBlueprint.make({
  params: {
    path: '/my-plugin',
    routeRef: rootRouteRef,
    loader: () => import('./components/MyPage').then(m => <m.MyPage />),
  },
});
```

```tsx
// src/components/MyPage/MyPage.tsx
import { Content } from '@backstage/core-components';

export function MyPage() {
  return (
    <Content>
      {/* page body — NO Page, Header, or PageWithHeader wrappers */}
    </Content>
  );
}
```

The framework's `PageLayout` automatically renders `PluginHeader` from the plugin's `title` and `icon`. The page component should not include `Page`, `Header`, or `PageWithHeader` from `@backstage/core-components` — those are old-system constructs.

`title` and `icon` params on `PageBlueprint` are only needed when they should differ from the plugin's own values. If omitted, the plugin-level values are used.

### Page with subtitle / custom actions

When the page needs a subtitle or action buttons below the framework header, use `Header` from `@backstage/ui`:

```tsx
import { Header } from '@backstage/ui';
import { Content } from '@backstage/core-components';

export function MyPage() {
  return (
    <>
      <Header
        title="Subtitle or description"
        customActions={
          <>
            <CreateButton title="Create" to="/my-plugin/create" />
            <SupportButton>Help text</SupportButton>
          </>
        }
      />
      <Content>
        <MyPageContent />
      </Content>
    </>
  );
}
```

### Page that owns its layout (no framework header)

For home pages or dashboards that render their own layout end-to-end, set `noHeader: true` on the `PageBlueprint`:

```tsx
export const myDashboardPage = PageBlueprint.make({
  params: {
    path: '/my-plugin',
    routeRef: rootRouteRef,
    noHeader: true,
    loader: () =>
      import('./components/MyDashboard').then(m => <m.MyDashboard />),
  },
});
```

### Page that handles its own routing (drill-down detail pages)

For pages with detail routes (e.g. `/my-plugin/items/:id`), point the `loader` at a component that uses `<Routes>` internally:

```tsx
// src/extensions.tsx
export const myPage = PageBlueprint.make({
  params: {
    path: '/my-plugin',
    routeRef: rootRouteRef,
    loader: () => import('./components/Router').then(m => <m.MyPluginRouter />),
  },
});
```

```tsx
// src/components/Router.tsx
import { Routes, Route } from 'react-router-dom';

export function MyPluginRouter() {
  return (
    <Routes>
      <Route path="/" element={<ListPage />} />
      <Route path="/items/:id" element={<DetailPage />} />
    </Routes>
  );
}
```

For top-level tabs (Overview / Settings), use `SubPageBlueprint` instead — see `references/sub-pages.md`.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds `frontend-plugin`:

1. Create or replace `plugins/<id>/src/extensions.tsx` with a single `PageBlueprint` named after the plugin (e.g. `myPluginPage`), pointing at `rootRouteRef` from `./routes` and a `loader` that imports the page component.
2. Ensure the scaffold's example page component does NOT wrap its content in `Page`, `Header`, or `PageWithHeader`. If it does, strip those wrappers and replace with `<Content>...</Content>` (from `@backstage/core-components`).
3. Add the page extension to `src/plugin.tsx`'s `extensions` array.
4. The path should default to `/<id>` unless the user specified otherwise.

## Tests

Page extensions are tested by rendering the page component (not the extension) inside a test app:

```tsx
import { renderInTestApp } from '@backstage/frontend-test-utils';
import { screen } from '@testing-library/react';
import { MyPage } from './MyPage';

it('renders page content', async () => {
  renderInTestApp(<MyPage />);
  expect(await screen.findByText(/expected content/)).toBeInTheDocument();
});
```

For pages that use `useApi` with custom refs, wrap with `TestApiProvider` from `@backstage/frontend-test-utils` to mock dependencies:

```tsx
import {
  TestApiProvider,
  renderInTestApp,
} from '@backstage/frontend-test-utils';

renderInTestApp(
  <TestApiProvider apis={[[myPluginApiRef, mockApi]]}>
    <MyPage />
  </TestApiProvider>,
);
```
