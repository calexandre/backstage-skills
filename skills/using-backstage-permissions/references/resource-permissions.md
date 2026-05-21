# Pattern: Resource permissions

Use a `ResourcePermission` when you want the policy to express **rules and conditional decisions**. Basic permissions can only return ALLOW or DENY — they can't depend on which resource is being acted on. Resource permissions add two capabilities on top:

- **Rules** — named, parameterized conditions the policy author can mix and match (e.g. an `IS_OWNER` rule taking a `userRef` parameter).
- **Conditional decisions** — instead of resolving each potential result individually, the framework returns the rule/parameters back so you can translate them into a database-level filter. Critical for list endpoints where loading every candidate into memory would be wasteful.

This is more involved than basic permissions. Reach for it only when needed — see `references/declaring.md` for the basic case.

## When to use

Use a resource permission when:

- You want adopters to optionally set up custom rules in their app's `PermissionPolicy` (e.g. "owner can update", "team members can read", "anyone in project can comment"). The plugin author ships the resource type + a library of rules; the adopter's policy decides whether and how to use them.
- You want adopters to control list-filtering logic from their `PermissionPolicy` without modifying your plugin. With resource permissions, the policy can return a _conditional_ decision like "apply rule `IS_OWNER` with this user's ref". Your plugin translates that into a `WHERE` clause and the DB does the filtering. With basic permissions, the policy can only say ALLOW or DENY for the whole batch — so any "show user X only their own items" logic would have to live inside your route handler, hardcoded, and adopters couldn't customize who sees what without changing plugin code.

Stick with basic permissions when:

- The decision is the same for all instances of the resource type
- ALLOW/DENY is enough — "admins can do X", "anyone with role Y can do Z"
- There is no resource yet — "create" actions are always basic permissions, since the resource doesn't exist at the time of the check (there's nothing for rules to inspect or filter against). `read`/`update`/`delete` _can_ be resource permissions if you want per-instance rules, but they don't have to be — a plugin where "admins can read everything" stays basic for `read` even though there's a resource to point at.

## Standalone usage

### Declare the resource permission

In the `*-common` package (see `references/declaring.md`):

```ts
import { createPermission } from '@backstage/plugin-permission-common';

export const INCIDENT_RESOURCE_TYPE = 'incident';

export const incidentUpdatePermission = createPermission({
  name: 'incident.update',
  attributes: { action: 'update' },
  resourceType: INCIDENT_RESOURCE_TYPE,
});

export const incidentDeletePermission = createPermission({
  name: 'incident.delete',
  attributes: { action: 'delete' },
  resourceType: INCIDENT_RESOURCE_TYPE,
});
```

The `resourceType` string is what links the permission to the rules registered on the backend. Keep it stable — renaming it orphans existing policy bindings.

### Define rules and register the resource type (backend)

Rules describe possible conditions a policy can apply. Each rule has a name, a parameter schema, and **two methods** that express the same logical condition for two different runtime situations:

- **`apply(resource, params)`** runs when the framework already has the resource loaded in memory and needs to decide ALLOW/DENY for that single item. Used when a route handler calls `permissions.authorize(...)` with a specific `resourceRef` — the framework loads the resource via `getResources`, then calls `apply` to evaluate the rule. Returns a boolean.
- **`toQuery(params)`** runs when the framework returns a _conditional_ decision back to your plugin (from `authorizeConditional`) and the plugin needs to translate it into a database filter. Returns a query fragment in whatever shape your storage layer understands (here, a Knex `where` object; could be an SQL fragment, a Mongo query, an in-memory predicate generator, etc.). The plugin then uses this fragment to filter at the data layer instead of in-memory.

The two methods must agree: `apply(r, p) === true` for resource `r` if and only if `r` would be selected by `toQuery(p)`. Tests usually check both produce the same answer for the same data.

```ts
// src/permissions/rules.ts
import { makeCreatePermissionRule } from '@backstage/plugin-permission-node';
import { z } from 'zod';
import {
  INCIDENT_RESOURCE_TYPE,
  Incident,
} from '@your/plugin-incidents-common';

// `Incident` is the in-memory shape of a resource; `IncidentFilter` is the
// shape your data layer accepts (Knex where-object, MongoDB query, etc.).
const createRule = makeCreatePermissionRule<Incident, IncidentFilter>();

export const isOwnerRule = createRule({
  name: 'IS_OWNER',
  description: 'Allow when the current user is the incident owner',
  resourceType: INCIDENT_RESOURCE_TYPE,

  // The params the policy author passes when binding the rule.
  // For `IS_OWNER`, the policy says "use this rule with userRef = current user".
  paramsSchema: z.object({
    userRef: z.string(),
  }),

  // Single-resource check: given an already-loaded incident and the rule's
  // params, does it match? Used by authorize() with a resourceRef.
  apply(incident, { userRef }) {
    return incident.ownerRef === userRef;
  },

  // List-filter translation: given the rule's params, return a query fragment
  // that selects the same resources `apply` would return true for.
  // Used by authorizeConditional() in list endpoints.
  toQuery({ userRef }) {
    return { ownerRef: userRef };
  },
});

export const incidentRules = { isOwner: isOwnerRule };
```

Wire the rules into the module that registers your plugin:

```ts
// src/plugin.ts
import {
  createBackendPlugin,
  coreServices,
} from '@backstage/backend-plugin-api';
import {
  INCIDENT_RESOURCE_TYPE,
  incidentUpdatePermission,
} from '@your/plugin-incidents-common';
import { incidentRules } from './permissions/rules';

export const incidentsPlugin = createBackendPlugin({
  pluginId: 'incidents',
  register(env) {
    env.registerInit({
      deps: {
        permissionsRegistry: coreServices.permissionsRegistry,
        // ... other deps
      },
      async init({ permissionsRegistry /*, ... */ }) {
        permissionsRegistry.addResourceType({
          resourceRef: INCIDENT_RESOURCE_TYPE,
          permissions: [incidentUpdatePermission, incidentDeletePermission],
          rules: Object.values(incidentRules),
          getResources: async resourceRefs => {
            // Load incidents by ref. Order of returned array must match resourceRefs.
            return Promise.all(resourceRefs.map(ref => loadIncident(ref)));
          },
        });
        // ... rest of init
      },
    });
  },
});
```

The exact `addResourceType` shape is documented at `../backstage/packages/backend-plugin-api/src/services/definitions/PermissionsRegistryService.ts` — verify the field names (`resourceRef` vs `resourceType`, etc.) against the installed types in your project.

### Authorize per-resource in route handlers

In a handler that operates on a specific resource:

```ts
import { AuthorizeResult } from '@backstage/plugin-permission-common';
import { NotAllowedError } from '@backstage/errors';

router.patch('/incidents/:id', async (req, res) => {
  const credentials = await httpAuth.credentials(req, { allow: ['user'] });

  const [decision] = await permissions.authorize(
    [{ permission: incidentUpdatePermission, resourceRef: req.params.id }],
    { credentials },
  );
  if (decision.result !== AuthorizeResult.ALLOW) {
    throw new NotAllowedError('not allowed to update this incident');
  }

  // ... do the update
});
```

`authorize` resolves the conditional decision against the actual resource (using the `getResources` callback registered earlier) and returns ALLOW/DENY.

### Authorize a list query (the conditional path)

For endpoints that return a list, naive code calls `authorize` once per result — slow and chatty. Instead, use `authorizeConditional` to get the rule conditions back and translate them into a database filter:

```ts
import { AuthorizeResult } from '@backstage/plugin-permission-common';

router.get('/incidents', async (req, res) => {
  const credentials = await httpAuth.credentials(req, { allow: ['user'] });

  const [decision] = await permissions.authorizeConditional(
    [{ permission: incidentReadPermission }],
    { credentials },
  );

  if (decision.result === AuthorizeResult.DENY) {
    res.json({ items: [] });
    return;
  }

  if (decision.result === AuthorizeResult.ALLOW) {
    // No filter — caller can see everything.
    const items = await db('incidents').select('*');
    res.json({ items });
    return;
  }

  // CONDITIONAL — translate the conditions into a Knex query
  const filter = translateConditionsToKnex(decision.conditions, incidentRules);
  const items = await db('incidents').where(filter).select('*');
  res.json({ items });
});
```

`decision.conditions` describes which rules apply (with their parameters). Each rule's `toQuery(params)` produces a filter fragment; combine them with the framework helper `createConditionTransformer` (from `@backstage/plugin-permission-node`) instead of writing your own translator. Verify the helper name against `../backstage/plugins/permission-node/src/`.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a backend plugin AND intent describes per-resource access control ("only owner can edit", "users only see their team's items"):

1. Add `resourceType: '<resource-type>'` to the relevant permission(s) in the `*-common` package.
2. Create `plugins/<id>/src/permissions/rules.ts` with at least one rule scaffolded from the user's intent ("owner" → `IS_OWNER` rule querying an `ownerRef` column or annotation).
3. Add `permissionsRegistry: coreServices.permissionsRegistry` to the plugin's deps and call `permissionsRegistry.addResourceType(...)` in `init`.
4. In list handlers, switch from `authorize` to `authorizeConditional` and translate the decision into the data-layer filter.
5. Add `@backstage/plugin-permission-node` (for `makeCreatePermissionRule`) to `dependencies`.

## Tests

Test rules independently — they're pure functions:

```ts
import { mockCredentials } from '@backstage/backend-test-utils';
import { isOwnerRule } from './rules';

describe('isOwnerRule', () => {
  const incident = { id: '1', ownerRef: 'user:default/jdoe' } as any;

  it('matches when ownerRef equals userRef', () => {
    expect(isOwnerRule.apply(incident, { userRef: 'user:default/jdoe' })).toBe(
      true,
    );
  });

  it("doesn't match when refs differ", () => {
    expect(isOwnerRule.apply(incident, { userRef: 'user:default/other' })).toBe(
      false,
    );
  });

  it('produces a Knex-compatible filter', () => {
    expect(isOwnerRule.toQuery({ userRef: 'user:default/jdoe' })).toEqual({
      ownerRef: 'user:default/jdoe',
    });
  });
});
```

For integration tests, install a `mockServices.permissions(...)` factory that returns conditional decisions and assert your translator produces the right query:

```ts
import { AuthorizeResult } from '@backstage/plugin-permission-common';

// Mock returns CONDITIONAL with a known rule shape:
mockServices.permissions.factory({
  permissions: [
    {
      permission: incidentReadPermission,
      result: AuthorizeResult.CONDITIONAL,
      conditions: {
        rule: 'IS_OWNER',
        resourceType: INCIDENT_RESOURCE_TYPE,
        params: { userRef: 'user:default/jdoe' },
      },
    },
  ],
});
```

Then assert the route handler called the database with the filter `{ ownerRef: 'user:default/jdoe' }`.
