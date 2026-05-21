# Pattern: Defining a blueprint

`createExtensionBlueprint` from `@backstage/frontend-plugin-api` defines a typed factory other plugins call to contribute extensions of a particular shape. The blueprint declares:

- a **kind** — the type identifier (kebab-case)
- an **`attachTo`** — the parent extension and input the contributions feed into
- an **`output`** — the extension data refs every contribution must yield (required output)
- an optional **`configSchema`** — Zod schemas for app-config-driven per-contribution overrides
- an optional **`dataRefs`** — named refs the parent extension reads off each contribution
- a **`factory`** — a generator that yields the actual extension data values from the consumer's params

For the parent-side counterpart (declaring the input the blueprint attaches to), see `references/parent-collecting-extensions.md`.

## Standalone usage

### Step 1: Define data refs the contributions need

Data refs are the typed slots that flow between the contribution's factory and the parent's renderer. Some come from `coreExtensionData` (`reactElement`, `routePath`, etc.); others you define for plugin-specific data.

```ts
// src/blueprints/MetricsPanelBlueprint.ts
import { createExtensionDataRef } from '@backstage/frontend-plugin-api';

// A required-output ref: a sort key every panel must provide.
export const metricsPanelSortKeyDataRef = createExtensionDataRef<number>().with({
  id: 'metrics.panel.sort-key',
});

// An optional-output ref: a category label the panel may attach to.
export const metricsPanelCategoryDataRef = createExtensionDataRef<string>().with({
  id: 'metrics.panel.category',
});
```

Naming convention: `<plugin-id>.<concept>.<purpose>` — must be globally unique across the app.

### Step 2: Define the blueprint

```tsx
// src/blueprints/MetricsPanelBlueprint.ts (continued)
import {
  ExtensionBoundary,
  coreExtensionData,
  createExtensionBlueprint,
} from '@backstage/frontend-plugin-api';
import { z } from 'zod/v4';

export const MetricsPanelBlueprint = createExtensionBlueprint({
  kind: 'metrics-panel',

  // Where contributions attach. The id must match an extension in your plugin's
  // extensions array; the input must be declared on that extension.
  attachTo: { id: 'page:metrics', input: 'panels' },

  // The data refs every contribution must yield. coreExtensionData.reactElement
  // is what the parent renders.
  output: [
    coreExtensionData.reactElement,
    metricsPanelSortKeyDataRef,
    metricsPanelCategoryDataRef.optional(),
  ],

  // Named accessors the parent uses when reading contributions.
  dataRefs: {
    sortKey: metricsPanelSortKeyDataRef,
    category: metricsPanelCategoryDataRef,
  },

  // Adopter-overridable settings. Consumers can change these via app-config
  // without modifying their code.
  configSchema: {
    visible: z.boolean().optional(),
    sortKey: z.number().optional(),
  },

  // The factory: takes the params the consumer passed to .make(), plus the
  // runtime context (node, config, inputs, apis), and yields extension data.
  *factory(
    params: {
      loader: () => Promise<JSX.Element>;
      sortKey: number;
      category?: string;
    },
    { node, config },
  ) {
    // Honor adopter-overrides: config.sortKey wins over params.sortKey.
    const sortKey = config.sortKey ?? params.sortKey;

    yield coreExtensionData.reactElement(
      ExtensionBoundary.lazy(node, params.loader),
    );
    yield metricsPanelSortKeyDataRef(sortKey);
    if (params.category) {
      yield metricsPanelCategoryDataRef(params.category);
    }
  },
});
```

Key points:

- **`kind`** — appears in extension ids (`metrics-panel:my-plugin/foo`). Pick a stable, descriptive name; renaming is breaking.
- **`attachTo.id`** is the full id of the parent extension (`<kind>:<pluginId>` or `<kind>:<pluginId>/<name>`). Pluralize matches Backstage convention (`page:`, `entity-content:`, etc.).
- **`output`** — the array of data refs the framework verifies every contribution yields. Required refs must be yielded; mark optional refs with `.optional()` so contributors can skip them.
- **`ExtensionBoundary.lazy(node, loader)`** — wraps the consumer's loader in an error boundary and Suspense; standard for blueprints whose contributions render React.
- **`config.<field>` overrides `params.<field>`** is the convention. Consumers express defaults; adopters override via app-config.
- **The factory is a generator** (`function*` / `*factory(...)`). Each `yield ref(value)` adds one piece of extension data. The framework collects all yields and assembles the extension.
- **`config.has(...)` doesn't apply here** — the framework parses `configSchema` ahead of time, so `config` is already a typed object. Read fields directly.

### Step 3: Re-export from `src/index.ts`

```ts
// src/index.ts
export {
  MetricsPanelBlueprint,
  metricsPanelSortKeyDataRef,
  metricsPanelCategoryDataRef,
} from './blueprints/MetricsPanelBlueprint';
```

Other plugins import the blueprint to call `MetricsPanelBlueprint.make(...)`. They don't import the data refs directly — those are accessed via `MetricsPanelBlueprint.dataRefs.sortKey`.

### Function-form params (advanced)

For blueprints whose params include callbacks like `() => Promise<...>` (e.g. `FormFieldBlueprint` in upstream Backstage), use `createExtensionBlueprintParams` to give consumers type inference on the result:

```ts
import {
  createExtensionBlueprint,
  createExtensionBlueprintParams,
} from '@backstage/frontend-plugin-api';

// Without this, consumers writing `{ field: () => import('./MyField') }` lose
// inference into the imported module's exported type.
const defineParams = createExtensionBlueprintParams<{
  field: () => Promise<MyField>;
}>(/* ... */);

createExtensionBlueprint({
  kind: 'my-thing',
  attachTo: { /* ... */ },
  output: [/* ... */],
  defineParams,
  *factory(params, ctx) {
    const field = await params.field();
    // ...
  },
});
```

Verify the exact shape against `../backstage/packages/frontend-plugin-api/src/wiring/createExtensionBlueprint.ts` — the API has small surface variations across versions.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin` AND the user's intent explicitly describes "extensible plugin" or "let other plugins/modules contribute X":

1. Confirm the user actually needs a blueprint. Most plugins don't. If in doubt, ask: "Is anyone besides this plugin expected to contribute these?"
2. Create `plugins/<id>/src/blueprints/<Name>Blueprint.ts` (or co-located in `extensions.tsx` for one-off blueprints) with the `createExtensionBlueprint` call.
3. Define the data refs the blueprint requires. Naming: `<pluginId>.<thing>.<field>`.
4. Define the parent extension (see `references/parent-collecting-extensions.md`) in `extensions.tsx` with the matching `inputs.<input-name>` declaration.
5. Re-export the blueprint from `src/index.ts`.
6. Document for adopters in the plugin's README: import the blueprint, call `.make({ name, params })`, add to their plugin's `extensions` array.

## Tests

Blueprints are mostly type-level — the runtime behavior is exercised when the factory runs at app build time. Practical tests:

```tsx
import { startTestApp } from '@backstage/frontend-test-utils';
import { MetricsPanelBlueprint } from './MetricsPanelBlueprint';
import metricsPlugin from '../plugin';

it('a contribution registers and is reachable', async () => {
  const myPanel = MetricsPanelBlueprint.make({
    name: 'cpu',
    params: {
      loader: async () => <div>CPU panel</div>,
      sortKey: 10,
    },
  });

  const { findByText } = await startTestApp({
    features: [metricsPlugin],
    extensions: [myPanel],
  });

  expect(await findByText('CPU panel')).toBeInTheDocument();
});
```

Verify `startTestApp`'s exact API against `../backstage/packages/frontend-test-utils/`. For most blueprint correctness, type checking + a single happy-path integration test is sufficient — the framework owns most of the wiring logic.
