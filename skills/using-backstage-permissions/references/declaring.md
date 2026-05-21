# Pattern: Declaring permissions

Permissions are declared once in a `*-common` package so frontend and backend can both import them. The permission object is just data — `{ type, name, attributes }` — produced by `createPermission` from `@backstage/plugin-permission-common`.

## Standalone usage

### Where to declare

For a connected plugin (frontend + backend), place permissions in the shared `<id>-common` library — see `creating-backstage-plugin/SKILL.md` "Connected plugin sets". Both halves import the same permission object.

For a standalone plugin, declare in `src/permissions.ts` and re-export from `src/index.ts`.

### Basic permissions (action-style)

A basic permission represents a verb on a class of resource — "delete templates", "create incidents". The check is binary: does this user have it or not?

```ts
// plugins/<id>-common/src/permissions.ts
import { createPermission } from '@backstage/plugin-permission-common';

export const incidentsCreatePermission = createPermission({
  name: 'incidents.create',
  attributes: { action: 'create' },
});

export const incidentsReadPermission = createPermission({
  name: 'incidents.read',
  attributes: { action: 'read' },
});

export const incidentsUpdatePermission = createPermission({
  name: 'incidents.update',
  attributes: { action: 'update' },
});

export const incidentsDeletePermission = createPermission({
  name: 'incidents.delete',
  attributes: { action: 'delete' },
});

export const incidentsPermissions = [
  incidentsCreatePermission,
  incidentsReadPermission,
  incidentsUpdatePermission,
  incidentsDeletePermission,
];
```

Naming and attribute conventions:

- `name` is `<plugin-id>.<verb>` lowercase. Globally unique across the install — must not collide with another plugin's permission.
- `attributes.action` is one of `'create' | 'read' | 'update' | 'delete'` and helps app-policy authors apply consistent rules across similar permissions. It's optional but recommended.
- Export an `incidentsPermissions` array — the host app's policy code often iterates it to apply blanket rules.

### Resource permissions

A resource permission is parameterized by a specific instance — "can update _this_ incident". Required when the decision depends on which resource is being acted on (ownership, project membership, etc.).

```ts
// plugins/<id>-common/src/permissions.ts
import { createPermission } from '@backstage/plugin-permission-common';

export const INCIDENT_RESOURCE_TYPE = 'incident';

export const incidentUpdatePermission = createPermission({
  name: 'incident.update',
  attributes: { action: 'update' },
  resourceType: INCIDENT_RESOURCE_TYPE,
});
```

Adding `resourceType` makes the permission a `ResourcePermission<TResourceType>` rather than a `BasicPermission`. Resource permissions are used with `authorizeConditional` on the backend (see `references/resource-permissions.md`).

The `INCIDENT_RESOURCE_TYPE` string is the link between the permission and the rules registered against `permissionsRegistry` — keep it stable.

### Re-export from `src/index.ts`

```ts
export * from './permissions';
```

The plugin's frontend, backend, and any third-party consumers all import from this entry point.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a plugin AND intent describes access control:

1. If a `<id>-common` library exists (connected plugin set), put permissions there. If not, create `plugins/<id>/src/permissions.ts`.
2. Declare one `createPermission(...)` per CRUD verb the user mentioned. Default attributes:
   - "create X" / "add X" → `attributes: { action: 'create' }`
   - "view X" / "list X" / "see X" → `attributes: { action: 'read' }`
   - "edit X" / "modify X" → `attributes: { action: 'update' }`
   - "delete X" / "remove X" → `attributes: { action: 'delete' }`
3. If the user said "only the owner of X can…" or "for their own X", use `resourceType` and proceed to `references/resource-permissions.md`.
4. Add `@backstage/plugin-permission-common` to the relevant package's `dependencies`.

## Tests

Permission objects are constants — no tests needed. Type checking enforces correctness.

If you build helpers around a permissions array (e.g. a function that maps incident permissions to UI labels), unit-test the helper with the permission constants directly — they're plain objects.
