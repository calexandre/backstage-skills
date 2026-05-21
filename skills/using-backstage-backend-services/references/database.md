# Pattern: Database

`coreServices.database` returns a per-plugin Knex client. Each plugin gets an isolated schema/database, so plugins cannot accidentally read or write each other's tables.

## Standalone usage

### Getting a Knex client

```ts
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';

createBackendPlugin({
  pluginId: 'my-plugin',
  register(env) {
    env.registerInit({
      deps: {
        logger: coreServices.logger,
        database: coreServices.database,
      },
      async init({ logger, database }) {
        const knex = await database.getClient();

        await knex.migrate.latest({
          directory: resolvePackagePath('@your/scope-plugin', 'migrations'),
        });

        const items = await knex('items').select('*');
        logger.info(`loaded ${items.length} items`);
      },
    });
  },
});
```

Key points:

- `database.getClient()` is async — call it inside `init` (not at module top-level).
- The returned object is a standard Knex 3 instance (Backstage 1.50). All Knex idioms work: query builder, raw, transactions, schema builder.
- Per-plugin isolation: under SQLite this is a separate file; under Postgres a separate schema.

### Migrations

Place migration files in `src/migrations/` (or wherever you like) and run them in `init`:

```ts
// src/migrations/20260101000000-init.js
exports.up = async function (knex) {
  await knex.schema.createTable('items', table => {
    table.string('id').primary();
    table.string('name').notNullable();
    table.timestamp('created_at').defaultTo(knex.fn.now());
  });
};

exports.down = async function (knex) {
  await knex.schema.dropTable('items');
};
```

```ts
import { resolvePackagePath } from '@backstage/backend-plugin-api';

const migrationsDir = resolvePackagePath('@your/plugin-name', 'migrations');

await knex.migrate.latest({ directory: migrationsDir });
```

`resolvePackagePath` resolves a path inside a Node package without depending on `__dirname` — use it for any file resource that ships with your plugin.

### Transactions

```ts
await knex.transaction(async trx => {
  await trx('items').insert({ id: 'a', name: 'A' });
  await trx('items').insert({ id: 'b', name: 'B' });
});
```

## Post-scaffold usage

Only invoke if the user's intent mentions persistence ("stores X", "saves Y", "tracks Z over time", "history of W"). Otherwise skip.

When applicable:

1. Add `database: coreServices.database` to deps.
2. Inside `init`: `const knex = await database.getClient(); await knex.migrate.latest({ directory: resolvePackagePath('<plugin-package-name>', 'migrations') });`
3. Create `src/migrations/` directory with at least an `<timestamp>-init.js` migration creating the plugin's tables.
4. Add `knex` to `dependencies` in `package.json`.
5. Add `"files": ["dist", "migrations"]` to `package.json` so migrations ship with the published package.

## Tests

```ts
import { mockServices, TestDatabases } from '@backstage/backend-test-utils';

const databases = TestDatabases.create();

it.each(databases.eachSupportedId())(
  'creates the items table on migrate',
  async databaseId => {
    const knex = await databases.init(databaseId);
    const database = mockServices.database({ knex });

    // Run your code under test that uses `database`
    const client = await database.getClient();
    await client.migrate.latest({
      directory: resolvePackagePath('@your/plugin-name', 'migrations'),
    });

    const tables = await client('sqlite_master')
      .where({ type: 'table' })
      .select('name');
    expect(tables.map(t => t.name)).toContain('items');
  },
);
```

Notes:

- `TestDatabases.create()` provides isolated SQLite (and optionally Postgres) databases for each test. `databaseId` iterates the supported backends.
- `mockServices.database({ knex })` wraps a Knex instance into the `DatabaseService` shape so it can be passed to your code.
- For unit tests that don't need a real DB, prefer mocking the helper that wraps the Knex calls — Knex isn't friendly to inline mocking.
