# Intent detection: which patterns to pre-wire

Two categories of pattern skills get pre-wired during scaffolding:

1. **Always-on by template** — invoked regardless of intent because every plugin of that template needs them.
2. **Intent-driven** — invoked only when the user's description signals their need.

## Always-on by template

| Template                    | Skills to invoke unconditionally                                                                             |
| --------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `frontend-plugin`           | `using-backstage-frontend-system`, `using-backstage-ui`                                                      |
| `frontend-plugin-module`    | `using-backstage-frontend-system`, `using-backstage-ui`                                                      |
| `plugin-web-library`        | `using-backstage-ui`                                                                                         |
| `backend-plugin`            | `using-backstage-backend-system`, `using-backstage-backend-services`                                         |
| `backend-plugin-module`     | `using-backstage-backend-system`, `using-backstage-backend-services`                                         |
| `catalog-provider-module`   | `using-backstage-backend-system`, `using-backstage-backend-services`, `creating-backstage-catalog-provider`  |
| `scaffolder-backend-module` | `using-backstage-backend-system`, `using-backstage-backend-services`, `creating-backstage-scaffolder-action` |

For `plugin-node-library` and `plugin-common-library` templates, no automatic invocation — they're framework-agnostic. For `plugin-common-library` specifically, when scaffolded as part of a **connected plugin set**, populate `src/index.ts` with shared interfaces and types per the rules in `creating-backstage-plugin/SKILL.md` (Step 5, "connected plugin set" branch).

## Connected plugin sets

When the user describes a plugin that needs both a UI surface and its own dedicated backend (talks to an external system requiring server-side credentials, paginates large data, etc.), propose a 3-package set:

| Trigger signals                                                                                                                         | Package layout                                                                                     |
| --------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| External system mentioned (Jira, ServiceNow, PagerDuty, custom internal API) requiring auth tokens, secrets, or significant aggregation | `<id>` (frontend-plugin) + `<id>-backend` (backend-plugin) + `<id>-common` (plugin-common-library) |
| User explicitly names both halves ("Jira frontend AND backend")                                                                         | Same                                                                                               |
| User says "shows X from <system>" where X requires backend processing                                                                   | Same                                                                                               |

Don't propose the 3-package set when:

- The plugin only consumes existing Backstage backends (catalog, scaffolder, search, etc.)
- The plugin is presentational (forms, calculators, dashboards of static data)
- The user explicitly asks for just one half (e.g. "I just need the frontend")

## Intent-driven signals

Scan the user's description for additional patterns to pre-wire.

| Signal in description                                                                                                                  | Pattern skill to invoke                                                                                                                                                                                                           |
| -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "current user", "logged-in user", "my profile", "signed in", "for me"                                                                  | `using-backstage-identity` (frontend) or surface `coreServices.userInfo` via `using-backstage-backend-services` (backend)                                                                                                         |
| "my groups", "my teams", "my components", "things I own", "my ownership"                                                               | frontend: `using-backstage-identity` AND `using-backstage-catalog`. Backend: `using-backstage-catalog` plus `coreServices.userInfo` from `using-backstage-backend-services`.                                                      |
| "show entities", "list components", "table of services", "all groups", "filter by kind"                                                | `using-backstage-catalog`                                                                                                                                                                                                         |
| "fetch entity", "get the component", "lookup by ref", "specific entity"                                                                | `using-backstage-catalog` (fetch-by-ref)                                                                                                                                                                                          |
| "register a new", "add to catalog", "import location"                                                                                  | `using-backstage-catalog` (register-entity)                                                                                                                                                                                       |
| "stores in database", "persists", "saves to a table", "tracks history"                                                                 | flag `using-backstage-backend-services` (database section)                                                                                                                                                                        |
| "polls every", "runs every N minutes", "periodic sync", "scheduled"                                                                    | flag `using-backstage-backend-services` (scheduler section)                                                                                                                                                                       |
| "calls service-to-service", "auth token", "as the current user"                                                                        | flag `using-backstage-backend-services` (auth section)                                                                                                                                                                            |
| "fetches from GitHub/GitLab/Bitbucket", "reads file from URL", "imports yaml from a repo"                                              | flag `using-backstage-backend-services` (urlReader section)                                                                                                                                                                       |
| "caches", "memoizes", "key-value store"                                                                                                | flag `using-backstage-backend-services` (cache section)                                                                                                                                                                           |
| "scaffolder field", "custom field extension", "form field for templates", "ui:field", "custom picker"                                  | `creating-backstage-scaffolder-field` (typically inside a `frontend-plugin-module`)                                                                                                                                               |
| "only admins can…", "require permission", "must have permission to", "only owner can", "hide unless allowed", "gate behind permission" | `using-backstage-permissions` — frontend (`usePermission` / `<RequirePermission>`) and/or backend (`coreServices.permissions.authorize`). For "owner can" / per-resource decisions, also flag the resource-permissions reference. |
| "card on the entity page", "tab on the entity page", "show <thing> on the component / API / system page", "extend the catalog entity page", "entity overview card" | `creating-backstage-entity-page-extension` (typically inside a `frontend-plugin-module`) |
| "I want my plugin to be extensible", "let other plugins / modules contribute X to my plugin", "expose an extension point on the frontend", "create a blueprint for X" | `creating-backstage-extension-blueprint` (advanced — most plugins don't need this; confirm with the user first) |

(Add more rows as new pattern skills ship: `creating-backstage-custom-api`, etc.)

## Disambiguation

If the description is too vague to suggest patterns ("a plugin that does stuff"), do not pre-wire anything beyond the always-on set. Rely on the post-scaffold "offer to extend" step to surface options.

## Confirmation phrasing

When confirming with the user, translate skill names into plain English:

- `using-backstage-frontend-system` → "new frontend system structure (routes, plugin definition, page extension)"
- `using-backstage-ui` → "Backstage UI design system (BUI components and styling)"
- `using-backstage-backend-system` → "backend plugin/module structure (createBackendPlugin/Module, service registration)"
- `using-backstage-backend-services` → "backend services (logger, http router, etc.)"
- `using-backstage-identity` → "current user identity (frontend)"
- `using-backstage-catalog` (fetch) → "fetching a specific catalog entity"
- `using-backstage-catalog` (query) → "querying catalog entities with filters"
- `using-backstage-catalog` (register) → "registering catalog locations"
- `creating-backstage-catalog-provider` → "catalog entity provider (with EntityProvider class and module wiring)"
- `creating-backstage-scaffolder-action` → "custom scaffolder template action"
- `creating-backstage-scaffolder-field` → "custom scaffolder form field (UI component used by templates as ui:field)"
- `using-backstage-permissions` → "permission gating (declare, gate UI, authorize backend actions)"
- `creating-backstage-entity-page-extension` → "extension to the catalog entity page (card on Overview, or full tab)"
- `creating-backstage-extension-blueprint` → "expose your own blueprint so other plugins can extend this one"

The always-on skills can be implied rather than enumerated for brevity. Example for a single-package scaffold:

> I'll scaffold a `backend-plugin` named `metrics` with the standard backend system structure, and wire up:
>
> - querying catalog entities
> - scheduled background sync
>
> Sound right, or want to adjust?

Example for a **connected plugin set** (3 packages):

> The Jira plugin needs a UI plus a backend (the Jira API requires a server-side token), so I'll scaffold three packages:
>
> - `plugins/jira-common` — shared TypeScript types (`JiraProject`, `JiraIssue`, ...)
> - `plugins/jira-backend` — backend plugin with a `/projects/:key` route and a `JiraService` that calls the upstream Jira API
> - `plugins/jira` — frontend plugin with a `useJiraProject(key)` hook and a page that renders the result
>
> Sound right, or want to adjust (e.g. only need the frontend)?

## Edge cases

- **Frontend plugin needing only identity:** the always-on frontend skills still apply, plus invoke `using-backstage-identity`.
- **Backend plugin with no catalog or auth needs:** the always-on backend skills still apply (system + services). Don't add any optional skills.
- **Catalog provider:** `catalog-provider-module` template auto-invokes the foundational backend skills AND `creating-backstage-catalog-provider`. Don't separately invoke `using-backstage-catalog` (the provider doesn't typically read the catalog — it writes to it via the extension point).
- **Scaffolder action:** `scaffolder-backend-module` template auto-invokes the foundational backend skills AND `creating-backstage-scaffolder-action`.
- **Common library (`plugin-common-library`) and Node library (`plugin-node-library`):** never pre-wire patterns. These libraries are framework-agnostic.
- **Web library (`plugin-web-library`):** invoke `using-backstage-ui` only — these are React component libraries that should follow BUI conventions, but they don't define plugins or call catalog/identity APIs.
