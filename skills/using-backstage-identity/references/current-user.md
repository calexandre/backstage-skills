# Pattern: Get the current user

## Standalone usage

In any frontend component within an existing plugin:

```tsx
import React from 'react';
import useAsync from 'react-use/lib/useAsync';
import { useApi, identityApiRef } from '@backstage/frontend-plugin-api';
import { Skeleton, Alert, Text } from '@backstage/ui';

export const CurrentUserBanner = () => {
  const identityApi = useApi(identityApiRef);

  const { value, loading, error } = useAsync(async () => {
    const identity = await identityApi.getBackstageIdentity();
    const profile = await identityApi.getProfileInfo();
    return { identity, profile };
  }, [identityApi]);

  if (loading) return <Skeleton width={220} height={20} />;
  if (error)
    return (
      <Alert
        status="danger"
        title={`Failed to load identity: ${error.message}`}
      />
    );
  if (!value) return null;

  return (
    <Text>
      Signed in as {value.profile.displayName} ({value.identity.userEntityRef})
    </Text>
  );
};
```

Key points:

- `useApi(identityApiRef)` retrieves the API. Don't call it outside a component.
- `getBackstageIdentity()` returns `{ type, userEntityRef, ownershipEntityRefs }`.
- `getProfileInfo()` returns `{ displayName?, email?, picture? }`.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin`:

1. Open the scaffolded plugin's main page component (typically `plugins/<id>/src/components/ExampleComponent/ExampleComponent.tsx` — exact filename depends on the template version; locate the file with the `<PageHeader>` import).
2. Add the standalone snippet above as a new component in the same file (or a new file under `src/components/CurrentUserBanner/`).
3. Render `<CurrentUserBanner />` inside the page's `<Content>` block.
4. Verify imports — the scaffolded plugin already has `@backstage/core-plugin-api` as a dependency; no new package install needed.

## Tests

For frontend tests, use `@backstage/frontend-test-utils` — it ships `TestApiProvider`, `mockApis`, and a NFS-aware `renderInTestApp`:

```tsx
import {
  TestApiProvider,
  mockApis,
  renderInTestApp,
} from '@backstage/frontend-test-utils';
import { identityApiRef } from '@backstage/frontend-plugin-api';
import { screen } from '@testing-library/react';
import { CurrentUserBanner } from './CurrentUserBanner';

it('renders the signed-in user', async () => {
  renderInTestApp(
    <TestApiProvider
      apis={[
        [
          identityApiRef,
          mockApis.identity({
            userEntityRef: 'user:default/jdoe',
            ownershipEntityRefs: ['user:default/jdoe', 'group:default/team-a'],
            displayName: 'Jane Doe',
            email: 'jdoe@example.com',
          }),
        ],
      ]}
    >
      <CurrentUserBanner />
    </TestApiProvider>,
  );

  expect(await screen.findByText(/Jane Doe/)).toBeInTheDocument();
  expect(await screen.findByText(/user:default\/jdoe/)).toBeInTheDocument();
});
```

`mockApis.identity({...})` is the canonical NFS factory for identity test doubles — prefer it over hand-rolled mock objects.
