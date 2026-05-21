# Pattern: Auth, HTTP auth, user info

Three related services covering different auth needs:

- `coreServices.auth` — issue and verify tokens for service-to-service calls
- `coreServices.httpAuth` — authenticate incoming Express requests and read credentials
- `coreServices.userInfo` — resolve a user identity (entity ref, ownership refs) from credentials

## Standalone usage

### Service-to-service tokens (`coreServices.auth`)

When your plugin needs to call another Backstage backend plugin (e.g. catalog), mint a token via `auth`:

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
        auth: coreServices.auth,
        discovery: coreServices.discovery,
      },
      async init({ auth, discovery }) {
        const credentials = await auth.getOwnServiceCredentials();
        const { token } = await auth.getPluginRequestToken({
          onBehalfOf: credentials,
          targetPluginId: 'catalog',
        });

        const baseUrl = await discovery.getBaseUrl('catalog');
        const response = await fetch(
          `${baseUrl}/entities/by-name/component/default/example`,
          { headers: { Authorization: `Bearer ${token}` } },
        );
        if (!response.ok) {
          throw await ResponseError.fromResponse(response);
        }
        const entity = await response.json();
      },
    });
  },
});
```

Key methods:

- `auth.getOwnServiceCredentials()` — returns `BackstageCredentials` representing this plugin acting as itself.
- `auth.getPluginRequestToken({ onBehalfOf, targetPluginId })` — mints a token usable when calling another plugin.
- `auth.authenticate(token)` — verifies a token (rarely needed in plugin code; `httpAuth` handles incoming).

Pass the user's credentials as `onBehalfOf` instead of `getOwnServiceCredentials()` to mint a token that calls the target plugin **as the user**.

### Authenticating incoming requests (`coreServices.httpAuth`)

In a route handler, read credentials from the request:

```ts
env.registerInit({
  deps: {
    httpRouter: coreServices.httpRouter,
    httpAuth: coreServices.httpAuth,
  },
  async init({ httpRouter, httpAuth }) {
    const router = Router();

    router.get('/me', async (req, res) => {
      const credentials = await httpAuth.credentials(req, { allow: ['user'] });
      res.json({ principal: credentials.principal });
    });

    router.post('/internal-only', async (req, res) => {
      const credentials = await httpAuth.credentials(req, {
        allow: ['service'],
      });
      // credentials.principal.subject is the calling plugin's id
      res.json({ ok: true });
    });

    httpRouter.use(router);
  },
});
```

`allow` controls which principal types pass:

- `'user'` — a Backstage user (browser session or user token)
- `'service'` — another backend plugin
- `'none'` — unauthenticated (only works if `httpRouter.addAuthPolicy` allowed the route)

`credentials.principal` shape depends on the type:

- User: `{ type: 'user', userEntityRef: 'user:default/jdoe' }`
- Service: `{ type: 'service', subject: 'plugin:catalog' }`

### User info (`coreServices.userInfo`)

To go from credentials to richer user info (display name, ownership refs, profile):

```ts
env.registerInit({
  deps: {
    httpRouter: coreServices.httpRouter,
    httpAuth: coreServices.httpAuth,
    userInfo: coreServices.userInfo,
  },
  async init({ httpRouter, httpAuth, userInfo }) {
    const router = Router();

    router.get('/my-groups', async (req, res) => {
      const credentials = await httpAuth.credentials(req, { allow: ['user'] });
      const info = await userInfo.getUserInfo(credentials);
      res.json({
        userEntityRef: info.userEntityRef,
        ownershipEntityRefs: info.ownershipEntityRefs,
      });
    });

    httpRouter.use(router);
  },
});
```

`userInfo.getUserInfo(credentials)` returns `BackstageUserInfo`:

- `userEntityRef` — `'user:default/jdoe'`
- `ownershipEntityRefs` — array of refs the user is a member of (themselves + their groups)

## Post-scaffold usage

For `backend-plugin`:

- Include `httpAuth` only if intent describes user-authenticated routes (e.g. "the current user can…", "lists my…").
- Include `auth` only if intent describes calls to other Backstage plugins (e.g. "fetches from the catalog").
- Include `userInfo` if the plugin needs ownership refs (e.g. "filter by what the user owns").

A plain `backend-plugin` with only public routes doesn't need any of these.

## Tests

```ts
import {
  mockServices,
  mockCredentials,
  startTestBackend,
} from '@backstage/backend-test-utils';
import request from 'supertest';
import { myPlugin } from './plugin';

it('returns the principal at /me', async () => {
  const { server } = await startTestBackend({
    features: [
      myPlugin,
      mockServices.auth.factory(),
      mockServices.httpAuth.factory({
        defaultCredentials: mockCredentials.user('user:default/jdoe'),
      }),
      mockServices.userInfo.factory(),
    ],
  });

  const response = await request(server)
    .get('/api/my-plugin/me')
    .set('Authorization', mockCredentials.user.header('user:default/jdoe'));

  expect(response.status).toBe(200);
  expect(response.body.principal.userEntityRef).toBe('user:default/jdoe');
});
```

Notes:

- `mockCredentials.user(ref)`, `mockCredentials.service('plugin:foo')`, `mockCredentials.none()` produce credential objects.
- `mockCredentials.user.header(ref)` produces a header value to attach to test requests.
- `mockServices.httpAuth.factory({ defaultCredentials })` makes every authenticated request resolve to those credentials by default.
