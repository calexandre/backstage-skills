# Backstage Plugin Skills

A skill pack for [Claude Code](https://claude.com/claude-code) (and other [skills.sh](https://skills.sh)-compatible agents) that helps Backstage adopters create new plugins and apply common Backstage patterns idiomatically.

## Install

In any Backstage adopter repo:

```bash
npx skills add nosportugal/backstage-skills --skill '*'
```

Or pick individual skills interactively:

```bash
npx skills add nosportugal/backstage-skills
```

### As a Claude Code / GitHub Copilot CLI plugin

This repo is also a self-hosted plugin marketplace (`.claude-plugin/marketplace.json`),
so you can install the skill pack as a plugin straight from it:

```bash
# Claude Code
claude plugin marketplace add nosportugal/backstage-skills
claude plugin install backstage-skills@backstage-skills

# GitHub Copilot CLI
copilot plugin marketplace add nosportugal/backstage-skills
copilot plugin install backstage-skills@backstage-skills
```

## Usage

After install, skills activate automatically when your agent detects a matching intent:

- **Scaffold a plugin:** "Create a new Backstage plugin that shows catalog entities for the current user."
- **Apply a pattern in an existing plugin:** "Wire up the current user identity in this plugin."

The scaffolder asks for confirmation before running `yarn new` so you can adjust template, id, or pre-wired patterns.

## What's included

| Skill                                      | Purpose                                                                                                                                                                                                                                                                         |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `creating-backstage-plugin`                | Scaffold a new plugin via `yarn new`, auto-wiring NFS+BUI (frontend) or backend system (backend) defaults plus pattern skills based on intent.                                                                                                                                  |
| `using-backstage-frontend-system`          | Define frontend plugins: `createFrontendPlugin`, route refs, `ApiBlueprint`, `PageBlueprint`, `SubPageBlueprint`, NFS hooks.                                                                                                                                                    |
| `using-backstage-ui`                       | Build UI with the `@backstage/ui` design system: component catalog, CSS modules with BUI tokens, Remix Icons.                                                                                                                                                                   |
| `using-backstage-backend-system`           | Define backend plugins and modules: `createBackendPlugin`, `createBackendModule`, service registration, extension points.                                                                                                                                                       |
| `using-backstage-backend-services`         | Use `coreServices` (logger, config, http router, database, scheduler, lifecycle, cache, urlReader, discovery, auth, httpAuth, userInfo) idiomatically.                                                                                                                          |
| `using-backstage-identity`                 | Read the current logged-in user via `identityApiRef` (frontend).                                                                                                                                                                                                                |
| `using-backstage-catalog`                  | Interact with the catalog from frontend (`catalogApiRef`) or backend (`CatalogClient`): fetch by ref, query entities, register entities.                                                                                                                                        |
| `creating-backstage-catalog-provider`      | Build an `EntityProvider` and wire it into a catalog backend module via `catalogProcessingExtensionPoint`.                                                                                                                                                                      |
| `creating-backstage-scaffolder-action`     | Build a `createTemplateAction` and wire it into a scaffolder backend module via `scaffolderActionsExtensionPoint`.                                                                                                                                                              |
| `creating-backstage-scaffolder-field`      | Build a custom scaffolder form field (React component + Zod schema + optional validation) and register it via `FormFieldBlueprint`.                                                                                                                                             |
| `using-backstage-permissions`              | Declare permissions, gate frontend UI with `usePermission` / `<RequirePermission>`, authorize backend actions via `coreServices.permissions`, and (advanced) resource permissions with conditional decisions.                                                                   |
| `creating-backstage-entity-page-extension` | Add cards to the catalog entity's Overview tab via `EntityCardBlueprint`, or whole tabs via `EntityContentBlueprint`. Covers `useEntity`, entity filtering, and the `frontend-plugin-module` container.                                                                         |
| `creating-backstage-extension-blueprint`   | (Advanced) Define your OWN frontend blueprint via `createExtensionBlueprint` so other plugins/modules can contribute extensions to your plugin. Covers the parent extension that declares `inputs`, blueprint `factory` and `configSchema`, and `dataRefs` for consumer access. |

More pattern skills (custom API, frontend routing helpers, i18n, testing) will follow in subsequent releases.

## Compatibility

Patterns target current Backstage idioms. If APIs change in a major release, file an issue.

## Defaults

This pack assumes the new frontend system (NFS), the Backstage UI design system (BUI / `@backstage/ui`), and the new backend system (`@backstage/backend-plugin-api`) are the defaults for new plugins. Templates scaffolded via `creating-backstage-plugin` are auto-wired:

- **Frontend plugins** use `createFrontendPlugin`, `@backstage/frontend-plugin-api` route refs, and `@backstage/ui` for components. Styling is CSS modules with BUI design tokens; icons are `@remixicon/react`.
- **Backend plugins and modules** use `createBackendPlugin` / `createBackendModule`, depend on services from `@backstage/backend-plugin-api`, and consume the implementations provided by `@backstage/backend-defaults`.
- **Catalog providers** scaffold an `EntityProvider` class plus a module registering it against `catalogProcessingExtensionPoint`.
- **Scaffolder actions** scaffold a `createTemplateAction` plus a module registering it against `scaffolderActionsExtensionPoint`.

Pattern skills fall back to `@material-ui/core` only for the one known case where `@backstage/ui` doesn't have an equivalent (`@material-ui/lab` `Timeline`).

## Contributing

Each pattern skill follows the same shape:

1. `SKILL.md` with frontmatter (`name`, `description`) and a body that includes:
   - When to use this skill
   - A table mapping situations to reference files
   - Two-mode note (standalone vs post-scaffold)
   - Common gotchas
2. A `references/` folder where each `<pattern>.md` has three sections: **Standalone usage**, **Post-scaffold usage**, **Tests**.

To add a new pattern skill, copy an existing one (e.g. `using-backstage-identity`) and replace the references.
