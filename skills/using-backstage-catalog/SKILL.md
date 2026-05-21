---
name: using-backstage-catalog
description: Use when a Backstage plugin needs to interact with the catalog — fetching one entity by ref, querying entities with filters, or registering/unregistering catalog locations. Triggers on imports of catalogApiRef from @backstage/plugin-catalog-react, or @backstage/catalog-client in backend code.
---

# Backstage Catalog Patterns

## When to use this skill

- Plugin needs to fetch a specific entity by its ref (e.g. a Group, Component, or User)
- Plugin needs to query entities with filters (e.g. all Components owned by a team, all Groups of a given type)
- Plugin needs to register a new catalog location, refresh an entity, or unregister one programmatically
- This applies to both frontend and backend code; the entry points differ but the operations are the same

## Choose the right pattern

### Frontend (in a frontend plugin component)

| Situation                                  | Pattern                                                                      | Reference                                        |
| ------------------------------------------ | ---------------------------------------------------------------------------- | ------------------------------------------------ |
| Fetch one entity by ref                    | `useApi(catalogApiRef).getEntityByRef(ref)`                                  | references/fetch-by-ref.md (Frontend section)    |
| Query entities with filters                | `useApi(catalogApiRef).getEntities({ filter })`                              | references/query-entities.md (Frontend section)  |
| Register / refresh / unregister a location | `useApi(catalogApiRef).addLocation` / `refreshEntity` / `removeLocationById` | references/register-entity.md (Frontend section) |

### Backend (in a backend plugin or module)

| Situation                                  | Pattern                                                                                | Reference                                       |
| ------------------------------------------ | -------------------------------------------------------------------------------------- | ----------------------------------------------- |
| Fetch one entity by ref                    | `catalog.getEntityByRef(ref, { credentials })` via `catalogServiceRef`                 | references/fetch-by-ref.md (Backend section)    |
| Query entities with filters                | `catalog.getEntities({ filter }, { credentials })` via `catalogServiceRef`             | references/query-entities.md (Backend section)  |
| Register / refresh / unregister a location | `catalog.addLocation` / `refreshEntity` / `removeLocationById` via `catalogServiceRef` | references/register-entity.md (Backend section) |

## Two invocation modes

- **Standalone** (mid-development): user is editing an existing plugin. Pick the right reference based on the situation table above and apply.
- **Post-scaffold** (called by `creating-backstage-plugin`): apply the pattern to a freshly scaffolded plugin. Each reference's "Post-scaffold usage" section names the file and insertion point — frontend-template scaffolds use the Frontend pattern, backend-template scaffolds use the Backend pattern.

## Common gotchas

- In **frontend** code, `catalogApiRef` comes from `@backstage/plugin-catalog-react`, NOT `@backstage/catalog-client`. The latter is for backend.
- `useApi` itself comes from `@backstage/frontend-plugin-api` in NFS plugins; `catalogApiRef` is plugin-specific and stays in `@backstage/plugin-catalog-react`.
- In **backend** code, depend on `catalogServiceRef` from `@backstage/plugin-catalog-node` — it wraps the catalog client with auth/discovery wiring already done and takes `BackstageCredentials` directly. Methods take `{ credentials }` (a `BackstageCredentials` object), not `{ token }` (a string). For incoming HTTP requests, get credentials via `httpAuth.credentials(req)`. For service-to-service work (scheduled tasks, init-time fetches), use `auth.getOwnServiceCredentials()`. Don't manually `new CatalogClient(...)` — that's the underlying API but the service ref is the canonical surface.
- `getEntityByRef` returns `Entity | undefined`. Always handle the undefined case (entity may have been deleted or the ref may be malformed).
- `getEntities` returns `{ items: Entity[] }` — destructure or access `.items`.
- Filters on `getEntities` are AND-combined within an object, OR-combined across an array of objects.
- Frontend: don't fetch in a render — wrap in `useAsync` (auto-fetch on mount/dep change) or `useAsyncFn` (button-triggered fetch) from `react-use`. Don't manually wire `useState` for `{ loading, error, data }` — the hook handles race conditions and unmount cleanup correctly.
- Backend: don't fetch on every HTTP request without caching — for hot paths, use `coreServices.cache` to memoize results that don't need to be live.

## UI conventions

Use `@backstage/ui` for rendering when possible (`Skeleton`, `Alert`, `Text`, `Flex`, `Button`, …). See `using-backstage-ui` skill for the full component catalog and styling guidance.
