---
name: using-backstage-permissions
description: Use when a Backstage plugin needs to gate UI behind permissions (frontend) or authorize actions in route handlers (backend). Covers createPermission, the usePermission hook and RequirePermission component, the coreServices.permissions service for backend authorization, and resource permissions for per-instance checks. Triggers on phrases like "require permission", "gate this UI", "check authorization", or imports of @backstage/plugin-permission-common, @backstage/plugin-permission-react, or @backstage/plugin-permission-node.
---

# Backstage Permissions Patterns

## When to use this skill

- Plugin needs to hide a button, menu, or page from users without the right permission (frontend)
- Plugin's route handler needs to authorize the caller before performing an action (backend)
- Defining new permissions for a plugin
- Checking permissions per-resource ("can update _this_ component", not "can update components in general")

## Choose the right pattern

### Declaring permissions

| Building...                      | Pattern                                                          | Reference               |
| -------------------------------- | ---------------------------------------------------------------- | ----------------------- |
| A new permission for your plugin | `createPermission({ name, attributes })` in a `*-common` library | references/declaring.md |

### Frontend (in a plugin component)

| Situation                                     | Pattern                                                       | Reference              |
| --------------------------------------------- | ------------------------------------------------------------- | ---------------------- |
| Conditionally render UI based on a permission | `usePermission({ permission })` hook                          | references/frontend.md |
| Wrap an entire subtree in a permission gate   | `<RequirePermission permission={...}>...</RequirePermission>` | references/frontend.md |

### Backend (in a route handler or scheduled task)

| Situation                                           | Pattern                                                                                                                         | Reference                          |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| Authorize a basic action permission                 | `coreServices.permissions.authorize([{ permission }], { credentials })`                                                         | references/backend.md              |
| Authorize per-resource (with conditional decisions) | `permissions.authorizeConditional([{ permission, resourceRef }], { credentials })` + `permissionsRegistry.addResourceType(...)` | references/resource-permissions.md |

## Two invocation modes

- **Standalone** (mid-development): user is editing an existing plugin. Pick the right reference based on what's being added — defining a permission, gating UI, or authorizing on the backend.
- **Post-scaffold** (called by `creating-backstage-plugin`): when intent describes "only admins can…", "the current user must have permission to…", or any access-control language. Add the relevant patterns to the freshly scaffolded plugin.

## Common gotchas

- **Permissions are declared once, used twice.** The permission object lives in a `*-common` (or `*-permissions-common`) library so both the frontend (`usePermission`) and the backend (`permissions.authorize`) import the same definition. Don't redeclare the same permission in two places.
- **The host app must enable the permission framework.** Adopters set `permission.enabled: true` in `app-config.yaml` and install a `PermissionPolicy`. If permissions aren't enabled, `authorize` always returns `AuthorizeResult.ALLOW` — your plugin code is correct, just unenforced. Document the prerequisite in your plugin README.
- **`permissions.authorize` returns an array** matching the order of requests. For a single permission, destructure `const [decision] = await permissions.authorize(...)` — and the decision is `decision.result === AuthorizeResult.ALLOW`, NOT a boolean.
- **For resource permissions, prefer `authorizeConditional` over `authorize`** when the decision can depend on the resource (e.g. "owner can update"). It returns conditional decisions you translate into a database-level filter, avoiding the need to load every candidate resource into memory.
- **`usePermission` is async.** Returns `{ allowed, loading, error }` — render a `<Skeleton />` while loading, otherwise users see UI flicker as access is determined.
- **For backend plugins**, `coreServices.permissions` and `coreServices.permissionsRegistry` come from `@backstage/backend-plugin-api`. They're plugin-scoped — your plugin gets a `PermissionsService` that already knows its own pluginId.
