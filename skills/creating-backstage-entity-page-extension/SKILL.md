---
name: creating-backstage-entity-page-extension
description: Use when adding a card to a catalog entity's overview tab or a whole tab/page to a catalog entity page. Covers EntityCardBlueprint (overview-page cards) and EntityContentBlueprint (full tabs), entity filtering by kind/type/annotation, and access to the current entity via useEntity. Triggers on phrases like "card on the entity page", "tab on the component page", "show on entity overview", "ui:field for catalog entity page", or imports of EntityCardBlueprint / EntityContentBlueprint.
---

# Creating Catalog Entity Page Extensions

## When to use this skill

- Adding a card to the **Overview** tab of a catalog entity (e.g. "show Jira project info on Components linked to a Jira project")
- Adding a whole **tab** (a content panel with its own URL path) to a catalog entity page (e.g. "/incidents" tab on Components, listing the component's incidents)
- Filtering when an extension appears, based on entity kind, spec type, annotation presence, or any custom predicate

This is the most common reason to scaffold a `frontend-plugin-module`. The plugin you're authoring extends the host's catalog plugin via these blueprints.

## Choose the right pattern

| Building...                                      | Pattern                                                                                          | Reference             |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------ | --------------------- |
| A card on the entity Overview tab                | `EntityCardBlueprint.make({ name, params: { loader, filter?, type? } })`                         | references/card.md    |
| A full tab (its own URL path) on the entity page | `EntityContentBlueprint.make({ name, params: { path, title, loader, filter?, group?, icon? } })` | references/content.md |

Both blueprints come from `@backstage/plugin-catalog-react/alpha`. They share:

- A `loader: () => Promise<JSX.Element>` for code-splitting the rendered component.
- A `filter` parameter that controls when the extension appears (see "Entity filtering" below).
- The component renders inside the catalog entity context — read the current entity via `useEntity()` from `@backstage/plugin-catalog-react`.

## Two invocation modes

- **Standalone**: editing an existing `frontend-plugin-module` that targets the catalog. Pick the right reference (card or content) and add the extension.
- **Post-scaffold** (called by `creating-backstage-plugin` for `frontend-plugin-module` templates targeting `catalog`): scaffold the extension(s) the user described, plus the component(s) they render.

## Entity filtering

The `filter` parameter on both blueprints decides which entities the extension applies to. Three accepted forms:

1. **String** (most common): `'kind:component'`, `'kind:component,spec.type:service'`, `'metadata.annotations.acme.com/jira-project'`. Comma-separated for AND, multiple forms for the same key are OR.
2. **`FilterPredicate`** (structured): a Zod-validated object form for richer logic. Verify exact shape against `../backstage/packages/filter-predicates/src/`.
3. **Function**: `(entity: Entity) => boolean` — full programmatic control. Use sparingly — string filters compose better with the upstream config-override system.

If `filter` is omitted, the extension appears for all entity kinds. Almost always wrong — set a filter even if it's just `'kind:component'`.

## Common gotchas

- The blueprints come from `@backstage/plugin-catalog-react/alpha` (currently `@alpha`). Track the import path against `../backstage/plugins/catalog-react/src/alpha/blueprints/`.
- The component itself imports `useEntity` from `@backstage/plugin-catalog-react` (main entry, not `/alpha`). The hook returns `{ entity }` typed as `Entity` — use the catalog entity model from `@backstage/catalog-model` for typed access to `metadata`, `spec`, etc.
- `useEntity` only works inside a component rendered by one of these blueprints — calling it outside the catalog entity context throws.
- The `loader: () => Promise<JSX.Element>` form is required for code-splitting. Don't pass a synchronous component import — use `() => import('./MyCard').then(m => <m.MyCard />)`.
- For entity-specific data fetching, combine `useEntity` with the data-hook convention from `using-backstage-frontend-system` (a hook wrapping `useAsync` around `fetchApi.fetch`). Pass `entity.metadata.name` or an annotation as the cache key.
- The host app's catalog plugin owns the route `entity-content:catalog/overview` (where cards attach) and `page:catalog/entity` (where content tabs attach). Your module installs into the same backend as the catalog plugin — adopters install via `backend.add(...)` and `app.installPlugin(...)`; nothing else is needed.
