# Pattern: Parent extension that collects contributions

A blueprint is only half the story. The other half is the **parent extension** — usually a `PageBlueprint` extension your plugin owns — that declares an `inputs` slot and renders the contributions that attach to it.

Without a parent declaring the input, the contributions have nowhere to go. Without contributions, the parent shows an empty list. Both pieces ship together in your plugin.

## Standalone usage

### Step 1: Declare the input on the parent

Use the parent extension's blueprint with `makeWithOverrides` (which exposes `inputs`) instead of plain `make`:

```tsx
// src/extensions.tsx
import {
  PageBlueprint,
  createExtensionInput,
} from '@backstage/frontend-plugin-api';
import { rootRouteRef } from './routes';
import { MetricsPanelBlueprint } from './blueprints/MetricsPanelBlueprint';

export const metricsPage = PageBlueprint.makeWithOverrides({
  inputs: {
    panels: createExtensionInput([
      MetricsPanelBlueprint.dataRefs.sortKey,
      MetricsPanelBlueprint.dataRefs.category,
      // The framework also passes coreExtensionData.reactElement
      // automatically — no need to redeclare.
    ]),
  },
  factory(originalFactory, { inputs }) {
    return originalFactory({
      defaultPath: '/metrics',
      defaultTitle: 'Metrics',
      loader: async () => {
        // Sort contributions by their declared sortKey, then group by category.
        const panels = [...inputs.panels].sort(
          (a, b) =>
            a.get(MetricsPanelBlueprint.dataRefs.sortKey) -
            b.get(MetricsPanelBlueprint.dataRefs.sortKey),
        );

        return import('./components/MetricsPage').then(m => (
          <m.MetricsPage panels={panels} />
        ));
      },
    });
  },
});
```

Key points:

- **`inputs.panels`** — the input name MUST match the blueprint's `attachTo.input`. Mismatched names → silent no-attachments.
- **`createExtensionInput([...])`** — declares which data refs each contribution exposes via `.get(ref)`. List the refs the parent will read, including any `dataRefs` from the blueprint that the parent uses.
- **`makeWithOverrides`** — required to use `inputs`. Plain `.make()` doesn't accept inputs; `makeWithOverrides` gives you the `originalFactory` plus the resolved inputs.
- **Each input element** is an extension-data container. Read named refs via `.get(MyBlueprint.dataRefs.fieldName)`. The framework already wraps the React elements via `coreExtensionData.reactElement` — read those the same way.
- The parent passes the resolved contributions to the actual page component (here `MetricsPage`) so the page can decide how to render/order them.

### Step 2: Render the contributions in the page component

```tsx
// src/components/MetricsPage/MetricsPage.tsx
import { ReactNode } from 'react';
import { coreExtensionData } from '@backstage/frontend-plugin-api';
import { Content } from '@backstage/core-components';
import { Flex } from '@backstage/ui';
import { MetricsPanelBlueprint } from '../../blueprints/MetricsPanelBlueprint';

type Panel = {
  get<T>(ref: typeof coreExtensionData.reactElement): T;
  // Or use the typed shape produced by createExtensionInput inputs.
};

export function MetricsPage({ panels }: { panels: readonly Panel[] }) {
  return (
    <Content>
      <Flex direction="column" gap="4">
        {panels.map((panel, i) => {
          const element = panel.get(coreExtensionData.reactElement);
          const category =
            panel.get(MetricsPanelBlueprint.dataRefs.category) ?? 'general';
          return (
            <div key={i} data-category={category}>
              {element as ReactNode}
            </div>
          );
        })}
      </Flex>
    </Content>
  );
}
```

The page component receives the resolved contributions as a typed array and renders them in whatever shape it wants. Filtering, sorting, grouping all happen here — the framework just collects.

### Step 3: Wire both into the plugin

```tsx
// src/plugin.tsx
import { createFrontendPlugin } from '@backstage/frontend-plugin-api';
import { metricsPage } from './extensions';

export default createFrontendPlugin({
  pluginId: 'metrics',
  extensions: [metricsPage],
});
```

The blueprint itself isn't listed here — only the parent extension. The blueprint is exported from `src/index.ts` for consumer plugins to import and call `.make()` on.

### Default contributions from the same plugin

If your plugin ships some panels by default, list them in your `extensions` array alongside the parent:

```tsx
// src/extensions.tsx (continued)
import { MetricsPanelBlueprint } from './blueprints/MetricsPanelBlueprint';

export const cpuPanel = MetricsPanelBlueprint.make({
  name: 'cpu',
  params: {
    loader: () => import('./components/CpuPanel').then(m => <m.CpuPanel />),
    sortKey: 10,
    category: 'system',
  },
});

// src/plugin.tsx
export default createFrontendPlugin({
  pluginId: 'metrics',
  extensions: [metricsPage, cpuPanel],
});
```

Adopters can disable your defaults via app-config:

```yaml
app:
  extensions:
    - metrics-panel:metrics/cpu: false
```

## Post-scaffold usage

When scaffolding a plugin that exposes a blueprint:

1. Define the blueprint first (see `references/defining-the-blueprint.md`).
2. Add a parent extension to `plugins/<id>/src/extensions.tsx` using `<ParentBlueprint>.makeWithOverrides({ inputs: { <name>: createExtensionInput([...]) } })`. Match the input name to the blueprint's `attachTo.input`.
3. Implement the page component to receive `inputs.<name>` as a prop and render it.
4. List both the parent and any default contributions in `src/plugin.tsx`'s `extensions` array.
5. In the README, document for consumers: which blueprint to import, what params it accepts, what they need to do to add a contribution.

## Tests

Test the parent + blueprint pair by registering a contribution from inside a test app and asserting it renders:

```tsx
import { startTestApp } from '@backstage/frontend-test-utils';
import metricsPlugin from '../plugin';
import { MetricsPanelBlueprint } from '../blueprints/MetricsPanelBlueprint';

it('renders panels in sortKey order', async () => {
  const lowPriority = MetricsPanelBlueprint.make({
    name: 'low',
    params: {
      loader: async () => <div>Low</div>,
      sortKey: 100,
    },
  });
  const highPriority = MetricsPanelBlueprint.make({
    name: 'high',
    params: {
      loader: async () => <div>High</div>,
      sortKey: 1,
    },
  });

  const { findByText, getAllByText } = await startTestApp({
    features: [metricsPlugin],
    extensions: [lowPriority, highPriority],
  });

  await findByText('High');
  await findByText('Low');
  // Assert order:
  const allText = getAllByText(/Low|High/).map(n => n.textContent);
  expect(allText).toEqual(['High', 'Low']);
});
```

Notes:

- The test app instantiates your plugin (which has the parent extension) plus the test contributions; the framework wires them together.
- Use this style for: ordering, filtering, default-on/off behavior, app-config-driven overrides. For pure data-flow correctness, type checking does most of the work.
