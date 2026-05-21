---
name: creating-backstage-plugin
description: Use when the user wants to create a new Backstage plugin. Drives `yarn new` to scaffold the right template and optionally pre-wires common patterns (catalog access, identity, etc.) based on user intent. Triggers on phrases like "create a new plugin", "scaffold a Backstage plugin", "I want to add a frontend/backend plugin".
---

# Creating a Backstage Plugin

## When to use this skill

The user wants to bootstrap a new plugin in a Backstage repo. Either:

- They've described the plugin in free text ("I want a frontend plugin that lists my groups")
- Or they've asked generically ("scaffold a new plugin") and you should ask what they want

## The flow

Always follow these steps in order. Do not run `yarn new` until after the user confirms the plan in step 3.

### 1. Capture intent

If the user already described the plugin, parse it. Otherwise ask one question:

> "What kind of plugin do you want, and what should it do?"

### 2. Parse intent into a plan

Map the user's description to:

- **packages:** one or more package scaffolds. Most plugins are a single package, but **connected plugins** (a frontend that calls its own dedicated backend) need three: a `frontend-plugin`, a `backend-plugin`, and a `plugin-common-library` for shared types. See "Connected plugin sets" below for when to propose this.
- **template** (per package): one of the 9 supported templates (see `references/templates.md` for the full list and selection rules).
- **id** (per package): kebab-case. For connected sets, the three ids share a common base (e.g. `jira` + `jira-backend` + `jira-common`).
- **patterns** (per package): which pattern skills to invoke post-scaffold. See `references/intent-detection.md` for keyword → pattern heuristics.

#### Connected plugin sets

When the user describes a plugin that has **both** a UI surface AND its own dedicated backend (e.g. "a Jira plugin that shows project info" → it needs a backend to broker the Jira API call AND a frontend to render the result), propose a 3-package set instead of a single template:

| Package        | Template                | Purpose                                                                                     |
| -------------- | ----------------------- | ------------------------------------------------------------------------------------------- |
| `<id>`         | `frontend-plugin`       | The UI                                                                                      |
| `<id>-backend` | `backend-plugin`        | Backend routes; talks to the upstream system                                                |
| `<id>-common`  | `plugin-common-library` | Shared TypeScript types (request/response interfaces, domain models) imported by both sides |

Trigger signals (any of these → propose a 3-package set):

- The plugin description mentions an external system that needs an authenticated/server-side call (Jira, ServiceNow, PagerDuty, internal API not exposed to the browser)
- The user's description names a "page that shows X from <system>" where X requires backend computation, secrets, or pagination/aggregation that doesn't belong on the client
- The user explicitly asks for "frontend AND backend" or names both halves

Don't propose 3 packages when:

- The plugin only reads from already-installed Backstage backends (catalog, scaffolder, etc.) — the existing service refs cover those, no custom backend needed
- The plugin is purely presentational (form, calculator, dashboard of static data)

When proposing a 3-package set, list all three in the confirmation message in step 3 so the user can override (e.g. "actually I just need the frontend, the backend exists already"). If the user already has one of the three packages, scaffold only the missing ones.

### 3. Confirm with the user

Print a structured summary in this format and wait for explicit approval:

```
I'll scaffold a `<template>` named `<id>` and wire up:
- <pattern 1 description>
- <pattern 2 description>

Sound right, or want to adjust?
```

If the user wants changes, revise and re-confirm.

### 4. Run `yarn new`

Once confirmed, run from the repo root for each package in the plan. Single-package case:

```bash
yarn new --select <template> --option pluginId=<id>
```

Connected plugin set case — run three commands, in order (`-common` first so the others can depend on it):

```bash
yarn new --select plugin-common-library --option pluginId=<id>-common
yarn new --select backend-plugin --option pluginId=<id>
yarn new --select frontend-plugin --option pluginId=<id>
```

Note: `yarn new --select backend-plugin --option pluginId=<id>` produces `plugins/<id>-backend` (the `-backend` suffix is added by the template). The frontend produces `plugins/<id>`. The common produces `plugins/<id>-common`.

After each `yarn new`, verify the directory:

```bash
ls plugins/<id-or-suffix>
```

If any directory is missing, stop and surface the error to the user.

### 5. Apply patterns post-scaffold

For each pattern in the plan, invoke the corresponding pattern skill via the Skill tool. Pass the new plugin id and the directive that this is a post-scaffold invocation. Each pattern skill's references include a "Post-scaffold usage" section that names the file and insertion point.

**Frontend templates** (`frontend-plugin`, `frontend-plugin-module`):

- `using-backstage-frontend-system` — **always** invoke. Wires the canonical NFS structure (`src/routes.ts`, `src/plugin.tsx` with `createFrontendPlugin`, an example `PageBlueprint`).
- `using-backstage-ui` — **always** invoke. Replaces any MUI imports/styling the scaffold may have generated with `@backstage/ui` equivalents and CSS modules with BUI tokens.
- `using-backstage-catalog` — invoke (frontend mode) when intent involves catalog data.
- `using-backstage-identity` — invoke when intent involves the current user.
- `creating-backstage-scaffolder-field` — invoke for `frontend-plugin-module` when intent describes a scaffolder form field / `ui:field` / custom field extension.
- `creating-backstage-entity-page-extension` — invoke for `frontend-plugin-module` when intent describes a card or tab on the catalog entity page ("show X on the component page", "add a card for Y", "tab on the entity page").
- `using-backstage-permissions` — invoke (frontend mode) when intent describes gating UI behind permissions ("only admins see…", "require permission to…", "hide this button unless…").

**Frontend libraries** (`plugin-web-library`):

- `using-backstage-ui` — **always** invoke.

**Backend templates** (`backend-plugin`, `backend-plugin-module`):

- `using-backstage-backend-system` — **always** invoke. Wires the canonical structure (`src/plugin.ts` or `src/module.ts` using `createBackendPlugin`/`createBackendModule`, `src/index.ts` re-export).
- `using-backstage-backend-services` — **always** invoke. Adds `coreServices.logger` and (for plugins) `coreServices.httpRouter` as default deps; surfaces additional services if intent suggests them (database for persistence, scheduler for periodic work, auth for cross-plugin calls, etc.).
- `using-backstage-catalog` — invoke (backend mode) when intent involves catalog data.
- `using-backstage-permissions` — invoke (backend mode) when intent describes authorizing actions ("only admins can…", "must have permission to…", "owner-only…"). Adds `coreServices.permissions` to deps and `permissions.authorize` checks in route handlers.

**Specialist backend templates:**

- `catalog-provider-module` — also invoke `creating-backstage-catalog-provider` after the foundational backend skills.
- `scaffolder-backend-module` — also invoke `creating-backstage-scaffolder-action` after the foundational backend skills.

**Library templates** (`plugin-node-library`, `plugin-common-library`): no auto-invocation — these are framework-agnostic.

For a **connected plugin set**, after running the three `yarn new` commands:

1. Populate `plugins/<id>-common/src/index.ts` with the shared TypeScript types the plan calls for. What belongs in `-common`:

   - Domain interfaces and type aliases (e.g. `JiraProject`, `JiraIssue`)
   - Request/response shapes the backend route returns and the frontend hook consumes
   - Constants the backend route paths share with the frontend
   - Type predicates / type guards if useful
   - Zod schemas (if both sides validate the same shape)

   What does NOT belong in `-common`:

   - Runtime React or Express code
   - Anything that imports `@backstage/frontend-plugin-api` or `@backstage/backend-plugin-api`
   - Side-effecting initializers

2. Add `@your/plugin-<id>-common` (the package name from `<id>-common`'s `package.json`) to BOTH the frontend's and the backend's `dependencies`. Since this is a workspace, yarn resolves to the local package.
3. In the frontend's data hook (`useJiraProject`), import `JiraProject` from `@your/plugin-jira-common`.
4. In the backend's route handler / service, import the same type from the same package and use it as the route's response type.
5. Run `yarn install` from the repo root to wire the workspace links.

Then run the per-package post-scaffold flow above for the frontend and backend.

The pattern skill is the owner of its code samples — do not read its references directly.

### 6. Offer to extend

After patterns are applied, offer additional patterns the user didn't request initially:

> "Plugin scaffolded with <X, Y> wired up. Want to add anything else? (permissions, custom API, route, tests, …)"

If the user picks more, repeat step 5 for those skills.

### 7. Hand-off

Print a final summary:

> Plugin created at `plugins/<id>`. Run `yarn dev` to try it out.
> If you need to add more patterns later, invoke `using-backstage-frontend-system`, `using-backstage-backend-system`, `using-backstage-ui`, `using-backstage-backend-services`, `using-backstage-catalog`, `using-backstage-identity`, `using-backstage-permissions`, `creating-backstage-catalog-provider`, `creating-backstage-scaffolder-action`, `creating-backstage-scaffolder-field`, `creating-backstage-entity-page-extension`, or (advanced) `creating-backstage-extension-blueprint` directly.

## Common gotchas

- The id must be kebab-case. `yarn new` will reject `Foo` or `foo_bar`.
- Don't run `yarn new` interactively — always pass `--select` and `--option pluginId=`. Both flags are required for non-interactive use.
- If the repo isn't a Backstage repo (no `backstage.json` at root), surface that to the user before scaffolding.
- Each pattern skill expects a freshly scaffolded plugin in post-scaffold mode — the file structure may differ from older Backstage versions. Pattern skills handle the lookup; do not pass file paths to them.
