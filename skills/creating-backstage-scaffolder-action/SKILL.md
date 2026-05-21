---
name: creating-backstage-scaffolder-action
description: Use when building a custom scaffolder template action — a step that scaffolder templates can call from their YAML steps. Covers createTemplateAction with Zod input/output schemas, the action handler context (logger, workspacePath, input, output, etc.), and module wiring via scaffolderActionsExtensionPoint. Triggers on phrases like "scaffolder action", "template action", "custom action".
---

# Creating a Backstage Scaffolder Action

## When to use this skill

- Building a custom action that scaffolder templates can call (e.g. `acme:user:create`, `myorg:slack:notify`)
- Editing an existing `scaffolder-backend-module-*` plugin

## What you build

A scaffolder action is a backend module (`createBackendModule({ pluginId: 'scaffolder', ... })`) that registers a `TemplateAction` against the `scaffolderActionsExtensionPoint`. Two pieces:

1. The action itself (created via `createTemplateAction`)
2. The module that registers it

## Choose the right pattern

| Building...                 | Reference                         |
| --------------------------- | --------------------------------- |
| The action itself           | references/template-action.md     |
| The module that wires it in | references/module-registration.md |

## Two invocation modes

- **Standalone**: editing an existing scaffolder-backend-module-\*.
- **Post-scaffold** (called by `creating-backstage-plugin` for `scaffolder-backend-module`): scaffold both files together — action definition + module registration.

## Common gotchas

- Action `id` follows the `<namespace>:<name>` (or `<namespace>:<group>:<name>`) convention. The namespace is yours to choose; pick something stable that won't collide with other actions in the catalog of installed modules.
- Schemas are **factory functions**, not raw Zod schemas: `input: { name: z => z.string() }` (not `input: { name: z.string() }`). Backstage uses this pattern to pin its Zod version internally — verify the form against `../backstage/plugins/scaffolder-node/src/actions/createTemplateAction.ts`.
- Action handler context (`ctx`) provides: `ctx.logger`, `ctx.workspacePath`, `ctx.input` (typed from the input schema), `ctx.output(name, value)` to emit outputs, `ctx.getInitiatorCredentials()` for the user identity, `ctx.createTemporaryDirectory()` for sandboxed work, `ctx.signal` (AbortSignal) for cancellation, `ctx.isDryRun` for dry-run mode.
- Long-running actions should call `ctx.logger.info` periodically — the scaffolder UI streams logs to the user.
- For dry-run support, set `supportsDryRun: true` and check `ctx.isDryRun` in the handler to skip side effects.
