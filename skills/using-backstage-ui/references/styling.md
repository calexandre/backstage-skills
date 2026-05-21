# BUI Styling: CSS Modules + Design Tokens

BUI does not use `makeStyles` or theme objects. Component styling is done via CSS Modules colocated next to the component, referencing BUI's design tokens (CSS custom properties).

## Standalone usage

### File layout

```
src/components/MyComponent/
  MyComponent.tsx
  MyComponent.module.css
```

### CSS Module with BUI tokens

```css
/* MyComponent.module.css */
@layer components {
  .container {
    padding: var(--bui-space-4);
    background-color: var(--bui-bg-neutral-1);
    border-radius: var(--bui-radius-2);
  }

  .title {
    margin-bottom: var(--bui-space-2);
    color: var(--bui-fg-primary);
  }

  .row {
    display: flex;
    align-items: center;
    padding: var(--bui-space-2) 0;
  }
}
```

### Using the styles in a component

```tsx
import { Box, Text, Flex } from '@backstage/ui';
import { RiFileTextLine } from '@remixicon/react';
import styles from './MyComponent.module.css';

export function MyComponent() {
  return (
    <Box className={styles.container}>
      <Text variant="title-medium" className={styles.title}>
        Title
      </Text>
      <Flex className={styles.row} align="center">
        <RiFileTextLine size={24} />
        <Text>Content</Text>
      </Flex>
    </Box>
  );
}
```

### Token reference

Spacing:

| Token                | Approx pixel |
| -------------------- | ------------ |
| `var(--bui-space-1)` | 4px          |
| `var(--bui-space-2)` | 8px          |
| `var(--bui-space-3)` | 12px         |
| `var(--bui-space-4)` | 16px         |
| `var(--bui-space-6)` | 24px         |
| `var(--bui-space-8)` | 32px         |

Colors (foreground / background / border):

| Token                           | Use                       |
| ------------------------------- | ------------------------- |
| `var(--bui-fg-primary)`         | Primary text              |
| `var(--bui-fg-secondary)`       | Secondary text            |
| `var(--bui-fg-danger)`          | Error text                |
| `var(--bui-fg-info)`            | Link / info text          |
| `var(--bui-bg-app)`             | App background            |
| `var(--bui-bg-neutral-1)`       | Card / surface background |
| `var(--bui-bg-neutral-1-hover)` | Hover state               |
| `var(--bui-bg-solid)`           | Solid accent              |
| `var(--bui-ring)`               | Focus ring / accent       |
| `var(--bui-border-1)`           | Subtle border / divider   |

Radius:

| Token                    | Use             |
| ------------------------ | --------------- |
| `var(--bui-radius-2)`    | Small radius    |
| `var(--bui-radius-3)`    | Medium radius   |
| `var(--bui-radius-full)` | Full / circular |

Typography:

| Token                            | Use                 |
| -------------------------------- | ------------------- |
| `var(--bui-font-regular)`        | Default font family |
| `var(--bui-font-size-1)`         | Small font size     |
| `var(--bui-font-size-2)`         | Medium font size    |
| `var(--bui-font-size-3)`         | Large font size     |
| `var(--bui-font-weight-regular)` | Regular weight      |
| `var(--bui-font-weight-bold)`    | Bold weight         |

For most typography use the `<Text variant="…">` prop instead of styling text directly.

## Post-scaffold usage

If the scaffolded plugin's example component contains a `makeStyles` block:

1. Create a sibling `.module.css` file with the equivalent rules using BUI tokens (use the spacing/color/radius tables above to translate `theme.spacing(N)` and `theme.palette.*` references).
2. Wrap the rules in `@layer components { … }` so they participate in BUI's cascade ordering.
3. Replace the `makeStyles` import and `useStyles()` call with `import styles from './MyComponent.module.css'` and `className={styles.<name>}`.
4. Remove `@material-ui/core/styles` from the component file and from `package.json` if no longer used.

## Tests

Styling has no separate test pattern — component snapshot tests via `renderInTestApp` already cover class names. Visual regression should be handled by your CI screenshot tooling, not at the unit-test level.
