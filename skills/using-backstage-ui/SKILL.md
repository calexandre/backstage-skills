---
name: using-backstage-ui
description: Use when authoring UI for a Backstage frontend plugin. Covers the @backstage/ui (BUI) component library — layout, typography, forms, dialogs, menus, tables — plus styling via CSS modules with BUI design tokens, and Remix Icons. Triggers on imports of @backstage/ui or when creating any frontend component.
---

# Backstage UI (BUI) Patterns

## When to use this skill

- Authoring or editing any frontend component in a Backstage plugin
- Picking the right component for layout, typography, form input, dialog, menu, tab, table, etc.
- Styling components (BUI uses CSS modules + design tokens, not `makeStyles`)
- Adding icons (use `@remixicon/react`)

## Setup

A new plugin scaffolded with `yarn new` includes `@backstage/ui` automatically. If installing manually:

```bash
yarn add @backstage/ui @remixicon/react
```

Then add the stylesheet to your app entry (`packages/app/src/index.tsx` or similar):

```ts
import '@backstage/ui/css/styles.css';
```

## Choose the right component

| Need                   | Component                                                                                 | Reference                |
| ---------------------- | ----------------------------------------------------------------------------------------- | ------------------------ |
| Page layout / wrappers | `Box`, `Container`, `Flex`, `FullPage`, `Grid`                                            | references/components.md |
| Typography             | `Text`                                                                                    | references/components.md |
| Buttons / actions      | `Button`, `ButtonIcon`, `ButtonLink`                                                      | references/components.md |
| Forms                  | `TextField`, `PasswordField`, `SearchField`, `Select`, `Checkbox`, `RadioGroup`, `Switch` | references/components.md |
| Feedback               | `Alert`, `Skeleton`, `Tooltip`, `Tag`, `TagGroup`                                         | references/components.md |
| Overlays               | `Dialog`, `Menu`, `Popover`                                                               | references/components.md |
| Navigation             | `Tabs`, `Header`, `PluginHeader`, `Link`                                                  | references/components.md |
| Data                   | `Table` (+ `useTable` hook), `List` (+ `ListRow`), `Avatar`, `Card`                       | references/components.md |
| Icons                  | `@remixicon/react` (e.g. `RiSearchLine`, `RiAddLine`)                                     | references/icons.md      |
| Styling                | CSS modules with BUI tokens (`--bui-space-*`, `--bui-fg-*`, `--bui-bg-*`)                 | references/styling.md    |

## Two invocation modes

- **Standalone** (mid-development): user is editing an existing plugin component. Pick the relevant reference and apply.
- **Post-scaffold** (called by `creating-backstage-plugin` for any frontend template): apply BUI defaults to the freshly scaffolded plugin's example component, ensuring imports come from `@backstage/ui` and styles use CSS modules with BUI tokens.

## Common gotchas

- BUI ≠ Material UI. Don't mix prop names: `disabled` → `isDisabled`; `variant="contained"` → `variant="primary"`.
- `Flex` uses `justify="between"` (NOT `space-between`); `gap` goes via inline `style` or BUI tokens.
- `Tooltip` requires `TooltipTrigger` as a parent.
- `TextField`'s `onChange` receives a string directly, not an event object.
- Two known fallbacks where BUI doesn't have an equivalent: `NavItemBlueprint`'s `icon` prop expects a MUI `IconComponent` type (Remix icons aren't compatible there); `@material-ui/lab` `Timeline` has no BUI replacement. Use MUI in those specific cases.

## Data fetching convention

Don't manually wire `useState` for loading/error/data. Use `react-use` hooks:

- `useAsync(fn, deps)` — runs the async function on mount and whenever deps change. Use for "load data when the component renders or its props change."
- `useAsyncFn(fn)` — returns `[state, callback]`. Use for "load data when the user does X" (button click, form submit, etc.).

Manual `useState` for `{ loading, error, value }` is an anti-pattern in this codebase: it duplicates logic the hook handles correctly (race conditions, unmount-during-fetch, etc.). The catalog and identity references show `useAsync`; the same applies anywhere data is fetched.

```tsx
import useAsyncFn from 'react-use/lib/useAsyncFn';
import { Button, Skeleton, Alert, Text } from '@backstage/ui';

export const FetchOnDemand = () => {
  const [{ loading, error, value }, fetch] = useAsyncFn(async (id: string) => {
    return await someApi.get(id);
  }, []);

  return (
    <>
      <Button onClick={() => fetch('foo')}>Fetch</Button>
      {loading && <Skeleton width={200} height={20} />}
      {error && <Alert status="danger" title={error.message} />}
      {value && <Text>{value.name}</Text>}
    </>
  );
};
```
