---
name: using-backstage-backend-services
description: Use when a Backstage backend plugin or module needs a core service — logger, config, http router, database, scheduler, lifecycle, cache, urlReader, discovery, auth, httpAuth, or userInfo. Triggers on imports of coreServices.* refs from @backstage/backend-plugin-api or any deps map in a plugin/module register block.
---

# Backstage Backend Services

## When to use this skill

- Plugin or module needs logging, configuration, HTTP routing, database, scheduling, caching, URL fetching, service discovery, or auth
- User mentions any specific `coreServices.*` ref
- Adding a new dep to a `register({ deps: {...} })` block

## Choose the right service

| Need                                      | Service                                                                  | Reference                             |
| ----------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------- |
| Logging                                   | `coreServices.logger` (plugin-scoped, pre-tagged with pluginId)          | references/logger-and-config.md       |
| Read app-config.yaml                      | `coreServices.rootConfig`                                                | references/logger-and-config.md       |
| Register HTTP routes                      | `coreServices.httpRouter`                                                | references/http-router.md             |
| Knex DB access                            | `coreServices.database` (plugin-scoped, isolated schema)                 | references/database.md                |
| Periodic / cron tasks                     | `coreServices.scheduler`                                                 | references/scheduler-and-lifecycle.md |
| Startup / shutdown hooks                  | `coreServices.lifecycle` (plugin) or `coreServices.rootLifecycle` (root) | references/scheduler-and-lifecycle.md |
| Service-to-service tokens                 | `coreServices.auth`                                                      | references/auth.md                    |
| Authenticate incoming requests            | `coreServices.httpAuth`                                                  | references/auth.md                    |
| Resolve current user from credentials     | `coreServices.userInfo`                                                  | references/auth.md                    |
| Read external URLs (GitHub, GitLab, etc.) | `coreServices.urlReader`                                                 | references/urlReader-and-discovery.md |
| Resolve a backend plugin's base URL       | `coreServices.discovery`                                                 | references/urlReader-and-discovery.md |
| Key-value cache                           | `coreServices.cache`                                                     | references/cache.md                   |

## Two invocation modes

- **Standalone**: adding a service dep to an existing plugin or module.
- **Post-scaffold** (called by `creating-backstage-plugin` for backend templates): include `coreServices.logger` and `coreServices.httpRouter` as defaults; surface additional services if intent suggests them (e.g. "stores X" → database; "polls every N minutes" → scheduler).

## Common gotchas

- All refs come from `@backstage/backend-plugin-api`. The actual implementations come from `@backstage/backend-defaults`, but plugins consume the refs only.
- Plugin-scoped services give each plugin an isolated instance — `coreServices.database` returns a Knex client scoped to a per-plugin schema; `coreServices.logger` is pre-tagged with `pluginId`. Don't try to share a Knex instance across plugins by passing it around — call cross-plugin via `coreServices.discovery` + HTTP instead.
- The `auth`, `httpAuth`, and `userInfo` services are easy to confuse. `auth` mints/verifies tokens for service-to-service calls; `httpAuth` authenticates an incoming Express request; `userInfo` resolves a user identity from credentials returned by `httpAuth`.
- `coreServices.scheduler` schedules durable tasks across replicas — don't use `setInterval` for production tasks.
- For tests: use `mockServices.<name>(...)` to get an instance for unit tests; use `mockServices.<name>.factory(...)` when passing to `startTestBackend({ features: [...] })`. Not every service has a factory — `httpRouter`, `cache`, `urlReader`, `lifecycle` are typically provided by the test backend's defaults.
- When calling another Backstage plugin via `discovery` + `auth` (cross-plugin HTTP), handle non-OK responses with `throw await ResponseError.fromResponse(response)` from `@backstage/errors`. It parses the response body for structured error info and produces a typed error other Backstage layers (logging, error reporting, frontend `<ResponseErrorPanel>`) understand. Never just `throw new Error(\`Failed: ${response.status}\`)`.
- Inside route handlers, throw typed errors from `@backstage/errors` (`NotFoundError`, `InputError`, `NotAllowedError`, `ConflictError`, `ServiceUnavailableError`, `ForwardedError`, etc.) — the default backend error middleware turns them into properly-formatted HTTP responses. Don't `res.status(...).json(...)` for error cases. See `references/http-router.md`.
