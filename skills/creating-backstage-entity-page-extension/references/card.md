# Pattern: Entity card via `EntityCardBlueprint`

A card renders on the entity's **Overview** tab inside the catalog entity page. Use it for compact, glanceable information about the entity (e.g. "Jira project status", "On-call schedule", "Latest deploy").

For full tabs with their own URL path, use `EntityContentBlueprint` (see `references/content.md`).

## Standalone usage

### The component

A regular React component that reads the current entity via `useEntity` and renders a BUI card:

```tsx
// src/components/JiraProjectCard/JiraProjectCard.tsx
import { useEntity } from '@backstage/plugin-catalog-react';
import {
  Card,
  CardHeader,
  CardBody,
  Skeleton,
  Alert,
  Text,
} from '@backstage/ui';
import { useJiraProject } from '../../hooks/useJiraProject';

export function JiraProjectCard() {
  const { entity } = useEntity();
  const projectKey = entity.metadata.annotations?.['acme.com/jira-project'];

  const { value, loading, error } = useJiraProject(projectKey);

  if (!projectKey) return null; // Guard: filter should prevent this, but be safe

  return (
    <Card>
      <CardHeader>Jira project</CardHeader>
      <CardBody>
        {loading && <Skeleton width={200} height={20} />}
        {error && <Alert status="danger" title={`Error: ${error.message}`} />}
        {value && (
          <Text>
            {value.name} ({value.key}) — {value.lead?.displayName ?? 'no lead'}
          </Text>
        )}
      </CardBody>
    </Card>
  );
}
```

Notes:

- `useEntity()` returns `{ entity }` where `entity` is the current `Entity` from the catalog. Read annotations, spec fields, and metadata from there.
- For data fetching, use the data-hook convention from `using-backstage-frontend-system` — a hook wrapping `useAsync` around `fetchApi.fetch`. Cache key includes the entity-derived parameter (here `projectKey`).
- Use BUI components — `Card`, `CardHeader`, `CardBody` are the standard wrappers. See `using-backstage-ui`.

### The blueprint extension

```tsx
// src/extensions.tsx
import { EntityCardBlueprint } from '@backstage/plugin-catalog-react/alpha';

export const jiraProjectCard = EntityCardBlueprint.make({
  name: 'jira-project',
  params: {
    loader: () =>
      import('./components/JiraProjectCard').then(m => <m.JiraProjectCard />),
    filter: 'kind:component,metadata.annotations.acme.com/jira-project',
  },
});
```

Key params:

- **`loader`** — required. `() => Promise<JSX.Element>` for code-splitting. Always use the dynamic import form.
- **`filter`** — strongly recommended. Without it, the card renders on every entity kind. The string form here means "kind=Component AND has the `acme.com/jira-project` annotation."
- **`type`** (optional) — the card's category. The catalog entity page groups cards by type (`'about'`, `'links'`, etc.). Verify the available enum values against `../backstage/plugins/catalog-react/src/alpha/blueprints/extensionData.ts`. Most cards omit this.

### Wiring into the plugin

```tsx
// src/plugin.tsx
import { createFrontendPlugin } from '@backstage/frontend-plugin-api';
import { jiraProjectCard } from './extensions';

export default createFrontendPlugin({
  pluginId: 'jira-entity-cards',
  extensions: [jiraProjectCard],
});
```

For a module that exposes multiple cards, list them all in `extensions`.

### Filter expressions in detail

```ts
// All Components
filter: 'kind:component';

// Components of type 'service'
filter: 'kind:component,spec.type:service';

// Any entity that has a specific annotation (regardless of kind)
filter: 'metadata.annotations.acme.com/jira-project';

// Either Components OR APIs
filter: 'kind:component | kind:api';
```

The exact syntax (separators, escapes, supported keys) is in `../backstage/packages/filter-predicates/src/` — verify when something doesn't behave as expected. For complex logic, prefer a function predicate over a stringly-typed expression.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin-module` and the user's intent describes adding a card to the entity page:

1. Create `plugins/<id>/src/components/<CardName>/<CardName>.tsx` with a `useEntity()`-driven component matching the structure above.
2. Create or extend `plugins/<id>/src/extensions.tsx` with one `EntityCardBlueprint.make({ name, params: { loader, filter } })` per card. Use kebab-case for the extension `name`.
3. Add the extensions to `src/plugin.tsx`'s `extensions` array.
4. Add `@backstage/plugin-catalog-react`, `@backstage/catalog-model`, `@backstage/ui`, and `react` to `dependencies`.
5. If the card's data depends on an annotation, document the annotation name in the plugin README so adopters know to add it to their entities.

## Tests

Test the component independently — render it inside a `TestApiProvider` with a stubbed `EntityProvider` that supplies the current entity:

```tsx
import { render, screen } from '@testing-library/react';
import { TestApiProvider } from '@backstage/frontend-test-utils';
import { EntityProvider } from '@backstage/plugin-catalog-react';
import { JiraProjectCard } from './JiraProjectCard';

const entity = {
  apiVersion: 'backstage.io/v1alpha1',
  kind: 'Component',
  metadata: {
    name: 'my-service',
    annotations: { 'acme.com/jira-project': 'PROJ' },
  },
  spec: {},
} as any;

it('renders the project for the linked annotation', async () => {
  // Mock useJiraProject either via jest.mock at the module level, or by
  // providing the underlying APIs (fetchApi, discoveryApi) through TestApiProvider.
  render(
    <TestApiProvider
      apis={
        [
          /* mock fetchApi + discoveryApi */
        ]
      }
    >
      <EntityProvider entity={entity}>
        <JiraProjectCard />
      </EntityProvider>
    </TestApiProvider>,
  );

  expect(await screen.findByText(/PROJ/)).toBeInTheDocument();
});

it('renders nothing when the entity lacks the annotation', () => {
  const noAnnotation = { ...entity, metadata: { name: 'my-service' } };
  const { container } = render(
    <EntityProvider entity={noAnnotation}>
      <JiraProjectCard />
    </EntityProvider>,
  );
  // The filter normally prevents this case at the blueprint level,
  // but the component's `if (!projectKey) return null` guard makes it safe.
  expect(container).toBeEmptyDOMElement();
});
```

Notes:

- `EntityProvider` (from `@backstage/plugin-catalog-react`) is the test seam for `useEntity`. Pass any entity object that satisfies your component's expectations.
- The blueprint's `filter` is enforced by the host app at runtime — your unit test of the component should verify component-level guards, not the filter itself.
- For the data hook (`useJiraProject`), prefer mocking it at the module level via `jest.mock('../../hooks/useJiraProject', ...)` rather than reconstructing the full API stack in every test.
