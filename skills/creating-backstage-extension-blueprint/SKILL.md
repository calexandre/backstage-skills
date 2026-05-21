---
name: creating-backstage-extension-blueprint
description: Use when authoring a Backstage frontend plugin that wants to expose its OWN extension point — a typed "blueprint" that other plugins or modules can call to contribute to your plugin's UI. Covers createExtensionBlueprint (defining the blueprint), the parent extension that collects contributions via inputs, factory functions, configSchema for adopter overrides, and dataRefs for consumer access. Triggers on phrases like "create a blueprint", "expose an extension point on the frontend", "let other plugins extend my page", or imports of createExtensionBlueprint from @backstage/frontend-plugin-api.
---

# Creating a Frontend Extension Blueprint

## When to use this skill

You're authoring a frontend plugin and want to give other plugins or modules a typed, structured way to contribute UI to it. Think:

- A dashboards plugin that wants modules to register custom dashboard panels
- A code-quality plugin that wants modules to register additional checks shown on a results page
- A reusable "feature flag" plugin where modules contribute UI controls per flag

This is the same pattern Backstage itself uses for `PageBlueprint`, `EntityCardBlueprint`, `EntityContentBlueprint`, `FormFieldBlueprint`, `NavItemBlueprint`. You're creating a new one of those *for your plugin*.

**Don't** create a blueprint when:

- A built-in blueprint already covers the case (use `EntityCardBlueprint` for catalog-page cards, `FormFieldBlueprint` for scaffolder fields, etc.)
- Your extension point only fires once and only your plugin will ever populate it — just write a regular extension and skip the blueprint indirection
- You're in a backend plugin — backend uses **extension points** (`createExtensionPoint`), not blueprints. See `using-backstage-backend-system/references/extension-points.md`

## What you build

A blueprint comes in two coupled pieces:

1. **The blueprint itself** (`createExtensionBlueprint`) — the typed factory other plugins call as `MyBlueprint.make({ name, params })` to contribute a single instance.
2. **A parent extension** that declares an `input` and renders or uses everything attached to it. This is usually a `PageBlueprint` extension you define in your plugin's `extensions.tsx`.

Both pieces are required and must agree on the input name and the data refs the contributions emit. Most plugins author both in the same `extensions.tsx` (or a colocated module).

## Choose the right pattern

| Building... | Reference |
|---|---|
| The blueprint itself (`createExtensionBlueprint` definition) | references/defining-the-blueprint.md |
| The parent extension that collects contributions via `inputs` | references/parent-collecting-extensions.md |

## Two invocation modes

- **Standalone**: editing an existing frontend plugin to add a new blueprint or refine an existing one.
- **Post-scaffold** (called by `creating-backstage-plugin` for `frontend-plugin`): rare — only when the user's intent explicitly describes "I want my plugin to be extensible by other modules". Most plugins don't need this and shouldn't have it.

## Common gotchas

- **Blueprints live on the frontend**. The backend equivalent is `createExtensionPoint` — different API, different package. If you want both extensible UI AND extensible backend behavior, create both, with separate symbols and separate documentation.
- **The `attachTo` of your blueprint MUST point at an input declared on a real extension** in the same plugin's `extensions` array. Mismatched id/input strings produce silent "no contributions" — the blueprint compiles, but nothing ever attaches.
- **Pick `output` carefully — these are the data refs every contribution must yield**. If a blueprint requires `coreExtensionData.reactElement` plus a custom data ref, every consumer's factory has to yield both. Adding required refs later is a breaking change for consumers.
- **Use `dataRefs` for optional, named accessors** the parent reads off contributions (e.g. "the contribution can optionally provide a sort key, accessed as `MyBlueprint.dataRefs.sortKey`"). Required output goes in `output`; optional/contribution-typed-data goes in `dataRefs`.
- **`configSchema`** lets adopters override per-contribution settings via app-config without touching code. Define one only when the contribution has user-facing settings (titles, paths, filters); skip otherwise.
- **The `factory` is a generator** — `function* (...)` yielding `ref(value)` calls. Each `yield` adds one piece of extension data. Loops, conditionals, and async-looking work all happen synchronously inside the factory; truly-async work belongs in the `loader` callback the params pass.
- **Two-stage params (`createExtensionBlueprintParams`)** — for blueprints whose params need to be a function-form (e.g. `{ field: () => Promise<...> }`), wrap with `createExtensionBlueprintParams` so consumers get type inference. See `FormFieldBlueprint` in `../backstage/plugins/scaffolder-react/src/next/blueprints/FormFieldBlueprint.tsx` for a real example.
- **`make` vs `makeWithOverrides`**: consumers normally call `MyBlueprint.make({ name, params })`. The `makeWithOverrides` variant lets a more advanced consumer override `factory` to wrap the original — used when `inputs:` are added on top of a blueprint. Document `make` in your README; mention `makeWithOverrides` only for advanced cases.
