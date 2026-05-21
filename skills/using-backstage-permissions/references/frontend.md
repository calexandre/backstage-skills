# Pattern: Frontend permission gating

Two ways to gate UI on a permission:

- `usePermission` hook — for fine-grained conditional rendering inside a component.
- `<RequirePermission>` wrapper — for gating a whole subtree (a page, a card, a section).

Both come from `@backstage/plugin-permission-react`. They consume `permissionApiRef` internally; the host app provides the implementation.

## Standalone usage

### `usePermission` hook

```tsx
import { usePermission } from '@backstage/plugin-permission-react';
import { Button } from '@backstage/ui';
import { incidentsCreatePermission } from '@your/plugin-incidents-common';

export const CreateIncidentButton = () => {
  const { allowed } = usePermission({
    permission: incidentsCreatePermission,
  });

  if (!allowed) return null;

  return <Button variant="primary">Create incident</Button>;
};
```

Returns:

- `allowed: boolean` — `true` if the user has the permission. `false` while loading too — so until the first resolution, the gated element is hidden.
- `loading: boolean` — true on initial fetch and on every re-check (the underlying SWR cache revalidates on focus, network reconnect, etc.).
- `error?: Error` — set if the permission check failed (network error, etc.).

Render guidance:

- Decide based on `allowed`, not `loading`. Just `if (!allowed) return null` and let the button appear once permission resolves. SWR (which `usePermission` uses internally) keeps the previous `allowed` value during revalidations, so the gated element stays stable across re-checks — no flicker as long as you don't render alternate UI on every `loading: true`.
- **Avoid rendering a Skeleton on `loading: true`.** `usePermission` re-checks periodically (focus, reconnect, interval) and treating each `loading` as initial load produces visible flicker on every revalidation.
- For whole-page gates where you do want a loading treatment on first paint, prefer `<RequirePermission>` (covered below) — it handles initial-load and denial UI internally. Note its default denial UI is a "Page Not Found" component, so for **non-page** gated areas (a card, a section) pass `errorPage={<></>}` to silently hide the content instead.
- Don't render the gated element with `disabled` — the user shouldn't see something they can't access. Hide it entirely instead.
- For "deny" cases, return `null` (or a friendlier "you don't have access" panel for whole-page gates).

### Resource permissions in the frontend

When the permission is a `ResourcePermission`, pass `resourceRef` so the policy can decide based on the specific resource:

```tsx
import { incidentUpdatePermission } from '@your/plugin-incidents-common';

export const EditIncidentButton = ({
  incidentRef,
}: {
  incidentRef: string;
}) => {
  const { allowed } = usePermission({
    permission: incidentUpdatePermission,
    resourceRef: incidentRef,
  });

  if (!allowed) return null;

  return <Button variant="secondary">Edit</Button>;
};
```

`resourceRef` is whatever stable identifier your resource type uses — an entity ref for catalog entities, a UUID for plugin-owned resources, etc. The backend's resource type registration (see `references/resource-permissions.md`) interprets it.

### `<RequirePermission>` wrapper

For gating a whole page or large subtree, the wrapper component is cleaner than scattering `usePermission` calls:

```tsx
import { RequirePermission } from '@backstage/plugin-permission-react';
import { incidentsReadPermission } from '@your/plugin-incidents-common';

export const IncidentsPage = () => {
  return (
    <RequirePermission permission={incidentsReadPermission}>
      <IncidentsList />
    </RequirePermission>
  );
};
```

By default `<RequirePermission>` renders a **"Page Not Found"** component when access is denied. That's right for whole-page gates (the user navigated to something they can't see) but wrong inside an existing page — you don't want a "Page Not Found" box appearing inside an otherwise valid page.

For non-page gates (a card, a section, a sidebar widget), pass `errorPage={<></>}` to silently hide the content on denial:

```tsx
<RequirePermission permission={incidentsReadPermission} errorPage={<></>}>
  <IncidentsCard />
</RequirePermission>
```

For NFS plugins, the default-behavior `<RequirePermission>` is most useful inside a `PageBlueprint` `loader` to gate the whole page. Verify the exact prop name (`errorPage` vs other names) against `../backstage/plugins/permission-react/src/components/` — the API has shifted between versions.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin` or `frontend-plugin-module` AND intent describes access control:

1. Add `@backstage/plugin-permission-react` to the plugin's `package.json` dependencies.
2. If the plugin has a `*-common` package (connected plugin set), depend on it from the frontend; import the permission objects from there. Otherwise import from `'../permissions'`.
3. For "create"/"edit"/"delete" buttons → wrap in `usePermission` checks that hide them when not allowed.
4. For whole pages that are admin-only → wrap the page component body with `<RequirePermission>`.

## Tests

Mock `permissionApiRef` via `TestApiProvider` to exercise both allowed and denied states:

```tsx
import { render, screen } from '@testing-library/react';
import { TestApiProvider } from '@backstage/frontend-test-utils';
import { permissionApiRef } from '@backstage/plugin-permission-react';
import { CreateIncidentButton } from './CreateIncidentButton';

const allowingApi = {
  authorize: jest.fn().mockResolvedValue([{ result: 'ALLOW' }]),
};
const denyingApi = {
  authorize: jest.fn().mockResolvedValue([{ result: 'DENY' }]),
};

it('renders the button when allowed', async () => {
  render(
    <TestApiProvider apis={[[permissionApiRef, allowingApi]]}>
      <CreateIncidentButton />
    </TestApiProvider>,
  );
  expect(
    await screen.findByRole('button', { name: /create incident/i }),
  ).toBeInTheDocument();
});

it('hides the button when denied', async () => {
  render(
    <TestApiProvider apis={[[permissionApiRef, denyingApi]]}>
      <CreateIncidentButton />
    </TestApiProvider>,
  );
  // Wait briefly for the async resolution, then assert absence.
  expect(
    screen.queryByRole('button', { name: /create incident/i }),
  ).not.toBeInTheDocument();
});
```

The permission API mock's `authorize` shape mirrors the real `PermissionApi.authorize` — it takes a request array and returns a same-length array of `{ result: 'ALLOW' | 'DENY' | 'CONDITIONAL' }`. For most frontend tests, plain `'ALLOW'`/`'DENY'` is enough.
