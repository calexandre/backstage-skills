# Pattern: URL reader and discovery

Two services for talking to other systems.

- `coreServices.discovery` — resolve a base URL for another Backstage backend plugin (cross-plugin calls)
- `coreServices.urlReader` — read content from external URLs (GitHub, GitLab, Bitbucket, raw HTTPS) using configured integrations

## Standalone usage

### Discovery (`coreServices.discovery`)

```ts
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';
import { ResponseError } from '@backstage/errors';

createBackendPlugin({
  pluginId: 'my-plugin',
  register(env) {
    env.registerInit({
      deps: {
        discovery: coreServices.discovery,
        auth: coreServices.auth,
      },
      async init({ discovery, auth }) {
        const baseUrl = await discovery.getBaseUrl('catalog');
        // e.g. http://localhost:7007/api/catalog

        const credentials = await auth.getOwnServiceCredentials();
        const { token } = await auth.getPluginRequestToken({
          onBehalfOf: credentials,
          targetPluginId: 'catalog',
        });

        const response = await fetch(`${baseUrl}/entities`, {
          headers: { Authorization: `Bearer ${token}` },
        });
        if (!response.ok) {
          throw await ResponseError.fromResponse(response);
        }
        const { items } = await response.json();
      },
    });
  },
});
```

Methods:

- `discovery.getBaseUrl(pluginId)` — internal/server-side base URL.
- `discovery.getExternalBaseUrl(pluginId)` — public URL, used when generating links shown to users.

Use `discovery` together with `auth.getPluginRequestToken` for cross-plugin calls. For the catalog specifically, use `CatalogClient` from `@backstage/catalog-client` which handles both internally — see `using-backstage-catalog`'s backend section.

### URL reader (`coreServices.urlReader`)

For reading content out of source-control hosts using configured `integrations`:

```ts
env.registerInit({
  deps: {
    urlReader: coreServices.urlReader,
  },
  async init({ urlReader }) {
    // Read a single file
    const response = await urlReader.readUrl(
      'https://github.com/owner/repo/blob/main/path/to/file.yaml',
    );
    const buffer = await response.buffer();
    const content = buffer.toString('utf-8');

    // Read a whole directory tree
    const tree = await urlReader.readTree(
      'https://github.com/owner/repo/tree/main/path/to/dir',
    );
    const files = await tree.files();
    for (const file of files) {
      const data = await file.content();
      // file.path, file.lastModifiedAt
    }

    // Search for files matching a glob
    const search = await urlReader.search(
      'https://github.com/owner/repo/blob/main/**/*.yaml',
    );
    for (const file of search.files) {
      // file.url, file.lastModifiedAt
    }
  },
});
```

Key points:

- Integrations (GitHub PAT, GitLab token, Bitbucket app password, etc.) are configured globally under `integrations.*` in `app-config.yaml`. The plugin doesn't need to know the credentials.
- `readUrl` and `readTree` URL forms differ: `readUrl` points at a file (`/blob/`), `readTree` at a directory (`/tree/`). The exact URL convention depends on the host.
- Returned `buffer()` gives a Node `Buffer` — convert to string explicitly if you need text.

## Post-scaffold usage

Invoke when intent describes:

- "calls another backend plugin", "fetches from catalog/auth/etc." → `discovery` (often paired with `auth`)
- "reads from GitHub/GitLab/Bitbucket", "imports yaml from a repo", "syncs files from a URL" → `urlReader`

For a plain `backend-plugin` with no external integrations, skip both.

## Tests

```ts
import { mockServices } from '@backstage/backend-test-utils';

const discovery = mockServices.discovery();
// In tests, the mock returns predictable URLs like 'http://localhost:0/api/<pluginId>'.

// For urlReader, there's no built-in mock factory — wrap urlReader in your
// own helper and mock the helper, or stub the response object inline:
const urlReader = {
  readUrl: jest.fn().mockResolvedValue({
    buffer: async () => Buffer.from('file contents'),
  }),
  readTree: jest.fn(),
  search: jest.fn(),
} as any;
```

For integration tests, `startTestBackend` provides a `discovery` automatically (URLs route to the test server). For `urlReader`, install your own factory or wrap in a helper at the plugin level.
