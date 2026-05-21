# BUI Component Patterns

Idiomatic usage for the components in `@backstage/ui`. Pick the relevant section.

## Standalone usage

### Layout

```tsx
import { Box, Container, Flex, Grid } from '@backstage/ui';

// Flex column with centered items
<Flex direction="column" align="center" justify="between">
  <Flex direction="row" style={{ gap: 'var(--bui-space-4)' }}>
    {children}
  </Flex>
</Flex>;

// CSS Grid
<Grid.Root columns={{ sm: '12' }} gap="6">
  <Grid.Item colSpan={{ sm: '12', md: '6' }}>{content}</Grid.Item>
</Grid.Root>;
```

`Flex` props: `direction="row"|"column"`, `align`, `justify` (use `"between"`, not `"space-between"`). `gap` is set via inline `style` with a BUI token or via `style={{ gap: 16 }}`.

### Typography

```tsx
import { Text } from '@backstage/ui';

<Text variant="title-large">Heading</Text>;
<Text variant="title-small">Subheading</Text>;
<Text variant="body-medium">Body text</Text>;
<Text variant="body-small" color="secondary">
  Secondary text
</Text>;
```

Valid `variant`s: `title-large`, `title-medium`, `title-small`, `title-x-small`, `body-large`, `body-medium`, `body-small`, `body-x-small`. Other props: `weight="regular"|"bold"`, `color="primary"|"secondary"|"danger"|"warning"|"success"`, `truncate`, `as` (semantic element).

### Buttons and actions

```tsx
import { Button, ButtonIcon, ButtonLink } from '@backstage/ui';
import { RiDeleteBinLine } from '@remixicon/react';

<Button variant="primary" isDisabled={loading} onClick={handleSubmit}>
  Submit
</Button>;

<ButtonIcon
  aria-label="delete"
  isDisabled={!canDelete}
  onPress={handleDelete}
  icon={<RiDeleteBinLine size={16} />}
  variant="secondary"
/>;
```

`Button` variants: `primary`, `secondary`, `tertiary`. Other props: `isDisabled`, `destructive`, `loading`. Use `ButtonIcon` for icon-only actions; it requires `aria-label`.

### Forms

```tsx
import {
  TextField,
  PasswordField,
  SearchField,
  Select,
  Checkbox,
  RadioGroup,
  Radio,
  Switch,
} from '@backstage/ui';

<TextField
  isRequired
  id="title"
  label="Title"
  value={value}
  onChange={newValue => setValue(newValue)}
/>;

<RadioGroup label="Visibility" value={vis} onChange={setVis}>
  <Radio value="public">Public</Radio>
  <Radio value="private">Private</Radio>
</RadioGroup>;
```

Note: `TextField`'s `onChange` receives the new string value directly, not an event. Use `isRequired` (not `required`).

### Feedback

```tsx
import { Alert, Skeleton, Tag, TagGroup } from '@backstage/ui';

<Alert
  status="danger"
  icon
  title="Error"
  description="Something went wrong."
/>;

<Skeleton width={200} height={24} />;

<TagGroup>
  <Tag size="small">Production</Tag>
  <Tag size="small">v1.2.0</Tag>
</TagGroup>;
```

`Alert.status`: `info`, `success`, `warning`, `danger`. Other props: `icon` (boolean or custom element), `isPending` (spinner instead of icon), `customActions`.

### Overlays — Tooltip, Dialog, Menu

Tooltip uses a trigger pattern:

```tsx
import { Tooltip, TooltipTrigger, Text } from '@backstage/ui';

<TooltipTrigger>
  <Text>Hover me</Text>
  <Tooltip>Tooltip content</Tooltip>
</TooltipTrigger>;
```

Dialog uses a trigger + structured children:

```tsx
import {
  Dialog,
  DialogTrigger,
  DialogHeader,
  DialogBody,
  DialogFooter,
  Button,
} from '@backstage/ui';

<DialogTrigger>
  <Button variant="primary">Open</Button>
  <Dialog isDismissable onOpenChange={onOpenChange}>
    <DialogHeader>Title</DialogHeader>
    <DialogBody>Body content here.</DialogBody>
    <DialogFooter>
      <Button variant="primary" onClick={onConfirm}>
        Confirm
      </Button>
      <Button variant="secondary" slot="close">
        Cancel
      </Button>
    </DialogFooter>
  </Dialog>
</DialogTrigger>;
```

Menu uses a trigger + items:

```tsx
import { ButtonIcon, Menu, MenuItem, MenuTrigger } from '@backstage/ui';
import { RiMore2Line } from '@remixicon/react';

<MenuTrigger>
  <ButtonIcon aria-label="more" icon={<RiMore2Line />} variant="secondary" />
  <Menu>
    <MenuItem onAction={handleAction}>Action</MenuItem>
  </Menu>
</MenuTrigger>;
```

### Navigation — Tabs

```tsx
import { Tabs, TabList, Tab, TabPanel } from '@backstage/ui';

<Tabs defaultSelectedKey="overview">
  <TabList>
    <Tab id="overview">Overview</Tab>
    <Tab id="settings">Settings</Tab>
  </TabList>
  <TabPanel id="overview">Overview content</TabPanel>
  <TabPanel id="settings">Settings content</TabPanel>
</Tabs>;
```

For plugin-level tabs in the new frontend system, prefer `SubPageBlueprint` over hand-rolled `Tabs` (see `using-backstage-frontend-system`). Use `Tabs` only for in-page sections.

### Data — List, Table, Card

```tsx
import {
  List,
  ListRow,
  Card,
  CardHeader,
  CardBody,
  CardFooter,
} from '@backstage/ui';
import { RiFileTextLine } from '@remixicon/react';

<List>
  <ListRow icon={<RiFileTextLine size={20} />} description="Description">
    Title
  </ListRow>
</List>;

<Card>
  <CardHeader>Component</CardHeader>
  <CardBody>Content</CardBody>
  <CardFooter>Footer actions</CardFooter>
</Card>;
```

For tables with sorting, filtering, and pagination, use `Table` together with the `useTable` hook (supports `complete`, `offset`, and `cursor` pagination modes).

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a frontend template, apply these defaults to the example component the scaffolder generated:

1. Replace any `@material-ui/core` and `@material-ui/icons` imports with `@backstage/ui` and `@remixicon/react` equivalents. Common substitutions:
   - `Typography` → `Text`
   - `Box display="flex"` → `Flex`
   - `Paper` → `Card`
   - `Chip` → `Tag`
   - `IconButton` → `ButtonIcon`
   - MUI icons → Remix icons (see `references/icons.md`)
2. If the scaffold added a `makeStyles` block, replace it with a colocated `.module.css` file using BUI tokens (see `references/styling.md`).
3. Confirm `@backstage/ui` and `@remixicon/react` are listed in the new plugin's `package.json` dependencies (the scaffold should already include `@backstage/ui`; add `@remixicon/react` if icons are used).
4. Confirm the app's entry file imports `@backstage/ui/css/styles.css` once globally — if not already present in `packages/app`, surface this as a follow-up step for the user.

## Tests

For small presentational components, prefer plain `render` from `@testing-library/react` — there's no need to spin up a test app:

```tsx
import { render, screen } from '@testing-library/react';
import { Alert } from '@backstage/ui';

it('renders the error', async () => {
  render(<Alert status="danger" title="Boom" />);
  expect(await screen.findByText('Boom')).toBeInTheDocument();
});
```

`useApi` does NOT require `renderInTestApp` — wrap the component in `TestApiProvider` from `@backstage/frontend-test-utils` and `render` is enough:

```tsx
import { render, screen } from '@testing-library/react';
import { TestApiProvider } from '@backstage/frontend-test-utils';

render(
  <TestApiProvider apis={[[myPluginApiRef, mockApi]]}>
    <MyComponent />
  </TestApiProvider>,
);
```

Reach for `renderInTestApp` only when the component (or one of its children) needs the test app context — i.e. uses `useRouteRef`, renders a `<Link>` that resolves a route ref, or otherwise depends on the router/app shell.

Most BUI components render their `title` / `children` text directly into the DOM, so `findByText` queries work as expected. For interactive overlays (`Tooltip`, `Dialog`, `Menu`) you may need `userEvent` to open them before asserting on contents.
