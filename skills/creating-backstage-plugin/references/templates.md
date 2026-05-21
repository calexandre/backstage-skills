# `yarn new` templates

The Backstage CLI exposes 9 templates via `yarn new --select <template>`. Pick the one matching the user's intent.

| Template                    | When to use                                                                                             | Output                                                             |
| --------------------------- | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `frontend-plugin`           | Standalone frontend page or feature with its own route.                                                 | `plugins/<id>` with React components, routable.                    |
| `frontend-plugin-module`    | Frontend extension that augments another frontend plugin (e.g. adds a card to the catalog entity page). | `plugins/<id>-module-<host>` — registers extensions, no own route. |
| `backend-plugin`            | Standalone backend plugin with its own HTTP routes.                                                     | `plugins/<id>-backend` with router, services.                      |
| `backend-plugin-module`     | Backend extension that adds capabilities to another backend plugin (e.g. an additional auth provider).  | `plugins/<id>-backend-module-<host>`.                              |
| `plugin-web-library`        | Reusable React library shared between plugins (no plugin metadata, no route).                           | `plugins/<id>` exporting components.                               |
| `plugin-node-library`       | Reusable Node.js library shared between backend plugins.                                                | `plugins/<id>-node` exporting utilities.                           |
| `plugin-common-library`     | Library of types/constants shared between frontend and backend plugins.                                 | `plugins/<id>-common`.                                             |
| `catalog-provider-module`   | Backend module that ingests entities into the catalog from an external source (e.g. Azure DevOps).      | `plugins/catalog-backend-module-<source>`.                         |
| `scaffolder-backend-module` | Backend module exposing custom scaffolder template actions (e.g. a new `publish:my-target` action).     | `plugins/scaffolder-backend-module-<id>`.                          |

## Selection rules (in order)

Apply the first rule that matches:

0. User describes a plugin that needs **both a UI and its own dedicated backend** (talks to an external system requiring server-side credentials, or aggregates/paginates data) → propose a **connected plugin set**: `frontend-plugin` + `backend-plugin` + `plugin-common-library`. See "Connected plugin sets" in `creating-backstage-plugin/SKILL.md`.
1. User says "catalog provider" or "ingest entities from X" → `catalog-provider-module`.
2. User says "scaffolder action" or "template action" → `scaffolder-backend-module`.
3. User says "module for the catalog/scaffolder/auth backend" or "extend the X backend" → `backend-plugin-module`.
4. User says "module that adds something to <existing frontend plugin>" → `frontend-plugin-module`.
5. User says "shared types" or "common library" → `plugin-common-library`.
6. User says "shared backend utilities" or "node library" → `plugin-node-library`.
7. User says "shared React components" or "web library" → `plugin-web-library`.
8. User says "backend plugin" or describes server-side endpoints → `backend-plugin`.
9. Default for any frontend description ("page", "shows X", "displays Y", "UI for Z") → `frontend-plugin`.

If multiple rules could apply, ask the user to disambiguate before running `yarn new`.
