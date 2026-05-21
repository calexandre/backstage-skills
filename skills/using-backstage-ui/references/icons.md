# Icons: `@remixicon/react`

BUI uses [Remix Icons](https://remixicon.com) via the `@remixicon/react` package. Import individual icons by name; tree-shaking keeps the bundle small.

## Standalone usage

```tsx
import { RiSearchLine, RiAddLine, RiDeleteBinLine } from '@remixicon/react';

<RiSearchLine size={20} />;
<RiAddLine size={16} />;
<RiDeleteBinLine size={20} color="var(--bui-fg-danger)" />;
```

Common props:

- `size` — pixel size (number)
- `color` — CSS color or BUI token
- `className` — for additional CSS

### Naming convention

`Ri<Word><Style>` where style is `Line`, `Fill`, or `Outline`. Use `Line` (outline) by default unless you specifically want filled icons.

### Common substitutions when migrating from `@material-ui/icons`

| MUI Icon        | Remix Icon           |
| --------------- | -------------------- |
| `Close`         | `RiCloseLine`        |
| `Search`        | `RiSearchLine`       |
| `Settings`      | `RiSettingsLine`     |
| `Add`           | `RiAddLine`          |
| `Delete`        | `RiDeleteBinLine`    |
| `Edit`          | `RiEditLine`         |
| `Check`         | `RiCheckLine`        |
| `Error`         | `RiErrorWarningLine` |
| `Warning`       | `RiAlertLine`        |
| `Info`          | `RiInformationLine`  |
| `ExpandMore`    | `RiArrowDownSLine`   |
| `ExpandLess`    | `RiArrowUpSLine`     |
| `ChevronRight`  | `RiArrowRightSLine`  |
| `ChevronLeft`   | `RiArrowLeftSLine`   |
| `Menu`          | `RiMenuLine`         |
| `MoreVert`      | `RiMore2Line`        |
| `Visibility`    | `RiEyeLine`          |
| `VisibilityOff` | `RiEyeOffLine`       |
| `Description`   | `RiFileTextLine`     |

For anything not in the table, browse https://remixicon.com to find the matching icon.

## Post-scaffold usage

When applying BUI defaults to a freshly scaffolded plugin:

1. If the scaffold imports `@material-ui/icons/<Name>` anywhere, replace each with the Remix equivalent from the table above.
2. Add `@remixicon/react` to the plugin's `package.json` dependencies.
3. Remove `@material-ui/icons` from `package.json` if no longer used in this plugin.

### Known type-compat fallback

`NavItemBlueprint` (from `@backstage/frontend-plugin-api`) takes an `icon` prop typed as MUI `IconComponent`. Remix icons are not type-compatible there. For the nav item icon specifically, keep using a MUI icon. Use Remix icons for everything else in the plugin.

## Tests

Icons render as SVG elements without text content, so `findByText` won't locate them. Use `data-testid` when an icon is the only thing identifying a control:

```tsx
<ButtonIcon
  aria-label="delete"
  data-testid="delete-btn"
  icon={<RiDeleteBinLine />}
  onPress={handleDelete}
/>
```

```tsx
const button = await findByTestId('delete-btn');
fireEvent.click(button);
```

For icons inside `ButtonIcon`, prefer querying by the `aria-label`:

```tsx
const button = await findByRole('button', { name: 'delete' });
```
