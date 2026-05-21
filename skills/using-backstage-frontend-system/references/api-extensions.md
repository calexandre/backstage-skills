# Pattern: API extensions with `ApiBlueprint`

Plugin APIs are declared as `ApiBlueprint`-based extensions and listed in the plugin's `extensions` array. The owning plugin is the only place an API can be overridden from — apps that want to swap your default implementation must do so via a module that targets your `pluginId`.

## Default: data hooks. Only use `ApiBlueprint` when you need overridability.

For most adopter-built plugins, `ApiBlueprint` is overkill. The default pattern is a hook that wraps `useAsync` (or `useAsyncFn`) directly around the fetch — no client class, no extension.

```ts
// src/hooks/useJiraProject.ts
import {
  useApi,
  fetchApiRef,
  discoveryApiRef,
} from '@backstage/frontend-plugin-api';
import { ResponseError } from '@backstage/errors';
import useAsync from 'react-use/lib/useAsync';

export function useJiraProject(key: string | undefined) {
  const fetchApi = useApi(fetchApiRef);
  const discoveryApi = useApi(discoveryApiRef);

  return useAsync(async () => {
    if (!key) return undefined;
    const baseUrl = await discoveryApi.getBaseUrl('jira');
    const response = await fetchApi.fetch(
      `${baseUrl}/projects/${encodeURIComponent(key)}`,
    );
    if (!response.ok) {
      throw await ResponseError.fromResponse(response);
    }
    return response.json() as Promise<JiraProject>;
  }, [fetchApi, discoveryApi, key]);
}
```

For event-triggered fetches (button click, form submit), use `useAsyncFn` and expose the trigger:

```ts
// src/hooks/useFetchJiraProject.ts
import {
  useApi,
  fetchApiRef,
  discoveryApiRef,
} from '@backstage/frontend-plugin-api';
import { ResponseError } from '@backstage/errors';
import useAsyncFn from 'react-use/lib/useAsyncFn';

export function useFetchJiraProject() {
  const fetchApi = useApi(fetchApiRef);
  const discoveryApi = useApi(discoveryApiRef);

  return useAsyncFn(
    async (key: string) => {
      const baseUrl = await discoveryApi.getBaseUrl('jira');
      const response = await fetchApi.fetch(
        `${baseUrl}/projects/${encodeURIComponent(key)}`,
      );
      if (!response.ok) {
        throw await ResponseError.fromResponse(response);
      }
      return response.json() as Promise<JiraProject>;
    },
    [fetchApi, discoveryApi],
  );
}
```

```tsx
// In a component:
const [{ loading, error, value }, fetch] = useFetchJiraProject();
return <Button onClick={() => fetch(input)}>Load</Button>;
```

If multiple endpoints share helper logic (error parsing, auth headers, retry), extract that into a private helper function in the same file or a `src/api/utils.ts`. Group several hooks under `src/hooks/jira.ts` if convenient. **A class would just be a renamed module** — without `ApiBlueprint` it doesn't share state across the app, so the indirection earns nothing.

### When to escalate to `ApiBlueprint`

Only when one or more of these is true:

- You're publishing an **open-source plugin** where adopters might reasonably want to swap the implementation (point at a self-hosted instance, change auth flow, mock for testing in their app).
- The API surface needs to be **overridable** by other plugins or apps via `createFrontendModule`.
- You want a single shared instance across the app, app-config-driven, and registered as a plugin extension (e.g. for cross-component caching that must survive component unmount).

`ApiBlueprint` adds extensibility, not functional capability. If you're not sure, stay with hooks — you can always upgrade later by lifting the hook's body into a class behind an `ApiBlueprint` without changing call sites that move from `useFetchX()` to `useApi(xApiRef)`.

## Standalone usage

### Create the API ref

```ts
// src/api.ts
import { createApiRef } from '@backstage/frontend-plugin-api';

export interface MyPluginApi {
  fetchSomething(id: string): Promise<Something>;
}

export const myPluginApiRef = createApiRef<MyPluginApi>().with({
  id: 'plugin.my-plugin.client',
  pluginId: 'my-plugin',
});
```

The builder form (`createApiRef<T>().with(...)`) is preferred because ownership is explicit via `pluginId` rather than parsed from the `id` string. The `id` must still be globally unique across the app — the `pluginId` is ownership metadata, not a namespace prefix.

### Implement the API

```ts
// src/api.ts (continued)
import { DiscoveryApi, FetchApi } from '@backstage/frontend-plugin-api';
import { ResponseError } from '@backstage/errors';

export class MyPluginClient implements MyPluginApi {
  constructor(
    private readonly options: {
      discoveryApi: DiscoveryApi;
      fetchApi: FetchApi;
    },
  ) {}

  async fetchSomething(id: string) {
    const baseUrl = await this.options.discoveryApi.getBaseUrl('my-plugin');
    const res = await this.options.fetchApi.fetch(`${baseUrl}/things/${id}`);
    if (!res.ok) {
      throw await ResponseError.fromResponse(res);
    }
    return res.json();
  }
}
```

### Expose the API as an extension

```ts
// src/apis.ts
import {
  ApiBlueprint,
  discoveryApiRef,
  fetchApiRef,
} from '@backstage/frontend-plugin-api';
import { myPluginApiRef, MyPluginClient } from './api';

export const myPluginApi = ApiBlueprint.make({
  params: defineParams =>
    defineParams({
      api: myPluginApiRef,
      deps: { discoveryApi: discoveryApiRef, fetchApi: fetchApiRef },
      factory: ({ discoveryApi, fetchApi }) =>
        new MyPluginClient({ discoveryApi, fetchApi }),
    }),
});
```

Then add `myPluginApi` to the plugin's `extensions` array (see `references/plugin-definition.md`).

### API ownership and override rules

Only the owning plugin (or a `createFrontendModule` targeting that plugin's `pluginId`) can replace the default implementation. Ownership is determined first by the explicit `pluginId` on the `ApiRef`, falling back to inference from the `ApiRef` `id` string:

- `plugin.<pluginId>.*` → owned by that plugin
- `core.*` → owned by the `app` plugin

If app adopters want to replace your plugin's default API implementation, they must create a `createFrontendModule` with `pluginId` matching your plugin. They cannot override it from a different plugin or from a generic `app` module.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `frontend-plugin`:

1. **Default — a data hook**. Create `plugins/<id>/src/hooks/use<DomainNoun>.ts` that takes the relevant input(s) as args and returns the result of `useAsync` (auto-fetch) or `useAsyncFn` (event-triggered). The hook calls `useApi(fetchApiRef)` and `useApi(discoveryApiRef)` directly. Components render by destructuring `{ loading, error, value }`. Group multiple related hooks in one file if convenient (`src/hooks/jira.ts`).
2. **Only escalate to `ApiBlueprint`** if the user's intent describes an open-source plugin, extensibility, or explicit overridability ("so adopters can swap implementations", "publishable plugin", "module overrides"). Don't introduce a client class as an intermediate step — without `ApiBlueprint` it adds no value.
3. If using `ApiBlueprint`: create `plugins/<id>/src/api.ts` with the API ref and client, `plugins/<id>/src/apis.ts` with the `ApiBlueprint.make` registration, and add the extension to `src/plugin.tsx`'s `extensions` array.

## Tests

API client tests use Jest mocks for `discoveryApi` and `fetchApi`:

```ts
import { MyPluginClient } from './api';

it('hits the discovered base url', async () => {
  const discoveryApi = {
    getBaseUrl: jest.fn().mockResolvedValue('http://x/api'),
  };
  const fetchApi = {
    fetch: jest
      .fn()
      .mockResolvedValue(new Response(JSON.stringify({ id: 'foo' }))),
  };
  const client = new MyPluginClient({ discoveryApi, fetchApi } as any);
  await client.fetchSomething('foo');
  expect(fetchApi.fetch).toHaveBeenCalledWith('http://x/api/things/foo');
});
```

Component tests that consume the API use `TestApiProvider` from `@backstage/frontend-test-utils` to provide a mock implementation under `myPluginApiRef`:

```tsx
import {
  TestApiProvider,
  renderInTestApp,
} from '@backstage/frontend-test-utils';
import { myPluginApiRef } from './api';

const mockApi = { fetchSomething: jest.fn().mockResolvedValue({ id: 'foo' }) };

renderInTestApp(
  <TestApiProvider apis={[[myPluginApiRef, mockApi]]}>
    <ComponentUnderTest />
  </TestApiProvider>,
);
```
