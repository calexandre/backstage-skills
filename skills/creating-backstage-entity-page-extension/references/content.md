# Pattern: Entity content tab via `EntityContentBlueprint`

A content tab adds a new top-level tab to a catalog entity's page, with its own URL path and content area. Use it for substantial per-entity content that doesn't fit on the Overview tab — incidents lists, deployment timelines, dashboards, "files" browsers, etc.

For compact glanceable info, use `EntityCardBlueprint` (see `references/card.md`).

## Standalone usage

### The component

A regular React component that reads the current entity and renders content. Don't include a page-level `Page`/`Header`/`PageWithHeader` wrapper — the catalog entity page provides the chrome. Use BUI components inside `<Content>` from `@backstage/core-components`:

```tsx
// src/components/IncidentsContent/IncidentsContent.tsx
import { Content } from '@backstage/core-components';
import { useEntity } from '@backstage/plugin-catalog-react';
import { Skeleton, Alert, Text, Flex } from '@backstage/ui';
import { useIncidents } from '../../hooks/useIncidents';

export function IncidentsContent() {
  const { entity } = useEntity();

  const { value, loading, error } = useIncidents(entity.metadata.name);

  return (
    <Content>
      {loading && <Skeleton width={400} height={120} />}
      {error && <Alert status="danger" title={`Error: ${error.message}`} />}
      {value && (
        <Flex direction="column" gap="2">
          {value.map(i => (
            <Text key={i.id}>{i.title}</Text>
          ))}
        </Flex>
      )}
    </Content>
  );
}
```

Notes:

- `useEntity()` returns the current entity. Use entity-derived input as the cache key in your data hook.
- The component renders inside the entity-page tab area. The router has already matched the tab path for you — you don't need a `<Routes>` here.

### The blueprint extension

```tsx
// src/extensions.tsx
import { EntityContentBlueprint } from '@backstage/plugin-catalog-react/alpha';

export const incidentsContent = EntityContentBlueprint.make({
  name: 'incidents',
  params: {
    path: '/incidents',
    title: 'Incidents',
    loader: () =>
      import('./components/IncidentsContent').then(m => <m.IncidentsContent />),
    filter: 'kind:component',
  },
});
```

Key params:

- **`path`** — required. Mounted under the entity URL: `/catalog/<namespace>/<kind>/<name>/incidents`.
- **`title`** — required. Shown in the tab strip.
- **`loader`** — required. Same `() => Promise<JSX.Element>` form as cards. Use a dynamic import for code-splitting.
- **`filter`** — strongly recommended. Same syntax as cards (string / `FilterPredicate` / function). Without it, the tab appears on every entity kind.
- **`group`** (optional) — string label to group multiple tabs under a parent menu in the tab strip. Default groups exist for common categories. Pass `false` to opt out of grouping.
- **`icon`** (optional) — string name from BUI's icon set, or a `ReactElement`. Shown next to the title.
- **`routeRef`** (optional) — bind this tab to a route ref so other plugins can link directly to it. Required if your plugin's `routes` map declares an external link to this tab.

### Wiring into the plugin

```tsx
// src/plugin.tsx
import { createFrontendPlugin } from '@backstage/frontend-plugin-api';
import { incidentsContent } from './extensions';

export default createFrontendPlugin({
  pluginId: 'incidents-entity-content',
  extensions: [incidentsContent],
});
```

For a module exposing multiple tabs, list them in `extensions`. They'll appear in the tab strip in extensions-array order, with adopter overrides via app config.

### Adopter-side configuration

Adopters who install the module get the tab automatically. They can override per-tab settings via app config without code changes:

```yaml
# app-config.yaml
app:
  extensions:
    - entity-content:incidents-entity-content/incidents:
        config:
          title: 'On-call'
          group: false
```

Verify the exact extension-id format against your installed `@backstage/frontend-plugin-api` types — extension ids are `<kind>:<pluginId>/<name>` and the `config` overrides are governed by the blueprint's `configSchema`.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin-module` and the user's intent describes adding a tab to the entity page:

1. Create `plugins/<id>/src/components/<TabName>Content/<TabName>Content.tsx` with a `useEntity()`-driven component matching the structure above. Wrap the body in `<Content>` from `@backstage/core-components`.
2. Create or extend `plugins/<id>/src/extensions.tsx` with one `EntityContentBlueprint.make({ name, params: { path, title, loader, filter } })` per tab. Kebab-case for `name`; lowercase, leading-slash for `path`; sentence-case for `title`.
3. Add the extensions to `src/plugin.tsx`'s `extensions` array.
4. Add `@backstage/plugin-catalog-react`, `@backstage/catalog-model`, `@backstage/core-components`, `@backstage/ui` to `dependencies`.
5. Document any required entity annotations in the plugin README.

## Tests

Same shape as cards — render the component inside an `EntityProvider`, mock the data hook:

```tsx
import { render, screen } from '@testing-library/react';
import { EntityProvider } from '@backstage/plugin-catalog-react';
import { IncidentsContent } from './IncidentsContent';

jest.mock('../../hooks/useIncidents', () => ({
  useIncidents: jest.fn().mockReturnValue({
    loading: false,
    error: undefined,
    value: [{ id: '1', title: 'DB outage' }],
  }),
}));

const entity = {
  apiVersion: 'backstage.io/v1alpha1',
  kind: 'Component',
  metadata: { name: 'my-service' },
  spec: {},
} as any;

it('renders the incidents list for the entity', async () => {
  render(
    <EntityProvider entity={entity}>
      <IncidentsContent />
    </EntityProvider>,
  );
  expect(await screen.findByText('DB outage')).toBeInTheDocument();
});
```

For tests that exercise the routing surface (deep-linking to the tab, navigating between tabs), use `renderInTestApp` from `@backstage/frontend-test-utils` and install the module — but most behavior is component-level and covered by the simpler `render` + `EntityProvider` pattern.
