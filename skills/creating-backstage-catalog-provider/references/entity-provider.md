# Pattern: EntityProvider class

There are two provider models to choose from. See the SKILL.md "Choose the model first: normal vs incremental" table for the decision rule. Default to **normal** for small/medium volumes; pick **incremental** when the source has thousands+ of entities and supports pagination.

## Normal `EntityProvider`

The provider class implements the `EntityProvider` interface from `@backstage/plugin-catalog-node`. It fetches the full set of entities and emits them via the connection's `applyMutation`.

### The class

Modeled on `plugins/catalog-backend-module-azure-devops-entity-provider/src/providers/AzureDevOpsEntityProvider.ts` in this repo:

```ts
import { Entity } from '@backstage/catalog-model';
import {
  EntityProvider,
  EntityProviderConnection,
} from '@backstage/plugin-catalog-node';
import {
  LoggerService,
  SchedulerServiceTaskRunner,
} from '@backstage/backend-plugin-api';

export interface MyEntityProviderConfig {
  // Anything the provider needs (auth tokens, base URLs, filters, etc.)
  apiUrl: string;
}

export class MyEntityProvider implements EntityProvider {
  private readonly config: MyEntityProviderConfig;
  private readonly logger: LoggerService;
  private readonly schedule: SchedulerServiceTaskRunner;
  private connection?: EntityProviderConnection;

  constructor(options: {
    config: MyEntityProviderConfig;
    logger: LoggerService;
    schedule: SchedulerServiceTaskRunner;
  }) {
    this.config = options.config;
    this.logger = options.logger.child({ target: this.getProviderName() });
    this.schedule = options.schedule;
  }

  getProviderName(): string {
    return 'my-entity-provider';
  }

  async connect(connection: EntityProviderConnection): Promise<void> {
    this.connection = connection;
    await this.schedule.run({
      id: this.getProviderName(),
      fn: () => this.refresh(),
    });
  }

  async refresh(): Promise<void> {
    if (!this.connection) {
      throw new Error('Not connected');
    }

    this.logger.info('refreshing entities from external source');

    const entities = await this.fetchEntities();

    await this.connection.applyMutation({
      type: 'full',
      entities: entities.map(entity => ({
        entity,
        locationKey: this.getProviderName(),
      })),
    });

    this.logger.info(`emitted ${entities.length} entities`);
  }

  private async fetchEntities(): Promise<Entity[]> {
    // Replace with real fetch logic
    const response = await fetch(this.config.apiUrl);
    const items: Array<{ name: string; owner?: string }> =
      await response.json();

    return items.map<Entity>(item => ({
      apiVersion: 'backstage.io/v1alpha1',
      kind: 'Component',
      metadata: {
        name: item.name,
        annotations: {
          'backstage.io/managed-by-location': `url:${this.config.apiUrl}`,
          'backstage.io/managed-by-origin-location': `url:${this.config.apiUrl}`,
        },
      },
      spec: {
        type: 'service',
        lifecycle: 'production',
        owner: item.owner ?? 'unknown',
      },
    }));
  }
}
```

Key points:

- `getProviderName()` returns a stable identifier. Naming convention: lowercase-hyphen, descriptive of the source system.
- `connect(connection)` stores the connection and triggers the scheduler. The scheduler fires the `refresh` method on a configurable cadence.
- `refresh()` fetches the current state from the external system and calls `applyMutation` with `type: 'full'` — meaning "this is the entire set this provider owns; reconcile."
- `locationKey` is per-entity-envelope and helps the catalog distinguish entities from this provider vs others.
- Required annotations: `backstage.io/managed-by-location` and `backstage.io/managed-by-origin-location` — both point at the source URL or a URI scheme of your choosing (e.g. `azure-devops:<org>`).

### Delta updates (alternative to full)

If your source system tells you what changed since the last sync, prefer delta updates:

```ts
await this.connection.applyMutation({
  type: 'delta',
  added: entitiesToAdd.map(entity => ({
    entity,
    locationKey: this.getProviderName(),
  })),
  removed: refsToRemove.map(entityRef => ({
    entityRef,
    locationKey: this.getProviderName(),
  })),
});
```

`removed` takes entity refs (`'component:default/foo'`), not full entities.

Note: this `type: 'delta'` mutation is a normal-provider feature for emitting incremental changes within a single fetch cycle. It is **not** the same as the `IncrementalEntityProvider` model below — that's a separate framework for streaming page-by-page from a paginated source.

## Incremental `IncrementalEntityProvider`

For sources with thousands+ of entities, implement `IncrementalEntityProvider<TCursor, TContext>` from `@backstage/plugin-catalog-backend-module-incremental-ingestion`. The framework drives iteration: it calls your `next(context, cursor?)` in a loop, persists the cursor in the catalog database, and handles batching, backoff, and resume-after-restart. You don't run your own scheduler.

### The class

```ts
import { Entity } from '@backstage/catalog-model';
import { DeferredEntity } from '@backstage/plugin-catalog-node';
import {
  IncrementalEntityProvider,
  EntityIteratorResult,
} from '@backstage/plugin-catalog-backend-module-incremental-ingestion';
import { LoggerService } from '@backstage/backend-plugin-api';

// Cursor: serializable state that lets you resume on the next page.
// For paginated APIs, this is typically the page token / offset / "after" id.
export interface MyCursor {
  pageToken?: string;
}

// Context: per-burst setup state — connections, prepared queries, etc.
// The framework tears it down between bursts.
export interface MyContext {
  client: MyExternalClient;
}

export interface MyIncrementalProviderConfig {
  apiUrl: string;
  apiToken: string;
  pageSize: number;
}

export class MyIncrementalProvider
  implements IncrementalEntityProvider<MyCursor, MyContext>
{
  constructor(
    private readonly config: MyIncrementalProviderConfig,
    private readonly logger: LoggerService,
  ) {}

  getProviderName(): string {
    return 'my-incremental-provider';
  }

  // around() opens a context for a burst, calls burst(context) which
  // internally iterates next() multiple times, then tears down.
  async around(burst: (context: MyContext) => Promise<void>): Promise<void> {
    const client = await MyExternalClient.connect({
      apiUrl: this.config.apiUrl,
      token: this.config.apiToken,
    });

    try {
      await burst({ client });
    } finally {
      await client.close();
    }
  }

  // next() returns one page and the cursor for the page after.
  // The framework saves the cursor in the DB; on the next iteration it
  // passes that cursor back. When `done: true` is returned, the framework
  // moves on to its rest period.
  async next(
    context: MyContext,
    cursor?: MyCursor,
  ): Promise<EntityIteratorResult<MyCursor>> {
    const result = await context.client.listItems({
      pageToken: cursor?.pageToken,
      pageSize: this.config.pageSize,
    });

    const entities: DeferredEntity[] = result.items.map(item => ({
      entity: this.toEntity(item),
      locationKey: this.getProviderName(),
    }));

    if (!result.nextPageToken) {
      return { done: true, entities };
    }

    return {
      done: false,
      entities,
      cursor: { pageToken: result.nextPageToken },
    };
  }

  private toEntity(item: { name: string; owner?: string }): Entity {
    return {
      apiVersion: 'backstage.io/v1alpha1',
      kind: 'Component',
      metadata: {
        name: item.name,
        annotations: {
          'backstage.io/managed-by-location': `url:${this.config.apiUrl}`,
          'backstage.io/managed-by-origin-location': `url:${this.config.apiUrl}`,
        },
      },
      spec: {
        type: 'service',
        lifecycle: 'production',
        owner: item.owner ?? 'unknown',
      },
    };
  }
}
```

Key points:

- **`around(burst)`** is called once per burst. Open expensive resources (DB connections, OAuth tokens) here, call `burst(context)`, and tear them down in a `finally`. The framework calls it repeatedly across bursts.
- **`next(context, cursor?)`** is the iterator step. Return `{ done: false, entities, cursor }` if more pages remain (the framework will call you again with this cursor); return `{ done: true, entities? }` when you've reached the end. The cursor is persisted by the framework — make sure it's a plain JSON-serializable object.
- **No `connect()` and no scheduler.** The framework wraps your provider with its own connection management and scheduling, driven by the `IncrementalEntityProviderOptions` (see `references/module-registration.md`).
- **Same entity shape and annotation requirements** as the normal provider — `backstage.io/managed-by-location`, etc.
- **Optional event handler:** if you set `eventHandler` on the class, the provider can also respond to incoming events via the events backend. Useful for reactive ingestion (e.g. webhook-driven). Out of scope for the basic pattern here.

### When to use `done: true` vs `done: false`

- `{ done: false, entities, cursor }` — there are more pages. The framework will call `next` again on its next iteration with this cursor.
- `{ done: true, entities? }` — you've reached the end of the source. The framework starts its rest period (configured via `restLength`) and then begins a fresh ingestion cycle, calling `next` with `cursor: undefined` to start from page 1 again.

If `next` throws, the framework treats it as a transient failure and retries with backoff (configured via `backoff` option).

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `catalog-provider-module`:

1. **First, decide normal vs incremental.** Use the SKILL.md "Choose the model first" table. If the user's intent describes a known-large source ("ingest from a system with tens of thousands of repos", "stream from <large data warehouse>"), pick incremental. If it sounds like a POC or small source ("a JSON file with my services", "100ish components"), pick normal. If unclear, **ask**: "Roughly how many entities does the source contain? Does the source support pagination (cursor / page-token / offset)?"
2. Create `plugins/<scaffolded-id>/src/providers/<Name>EntityProvider.ts` based on the chosen template above. Choose a class name from the scaffolded plugin id (e.g. `JsonFileEntityProvider` for a module called `catalog-backend-module-json-file`).
3. The `getProviderName()` value should be a stable name derived from the source system, not the plugin id — these may differ.
4. Pick the right entity `kind` for the use case: `Component` for services/repos/websites, `Group` for teams, `User` for people, `Resource` for cloud resources, `API` for API definitions, `System`/`Domain` for grouping.
5. Add dependencies to `package.json`:
   - **Normal:** `@backstage/plugin-catalog-node`, `@backstage/catalog-model`, `@backstage/backend-plugin-api`.
   - **Incremental:** `@backstage/plugin-catalog-backend-module-incremental-ingestion`, `@backstage/catalog-model`, `@backstage/backend-plugin-api`. (`@backstage/plugin-catalog-node` only if you also re-export shared types like `DeferredEntity`.)

## Tests

### Normal provider

Unit-test the class directly with a mocked connection and runner:

```ts
import { mockServices } from '@backstage/backend-test-utils';
import { MyEntityProvider } from './MyEntityProvider';

it('emits entities via the connection on refresh', async () => {
  const connection = {
    applyMutation: jest.fn(),
    refresh: jest.fn(),
  };

  // SchedulerServiceTaskRunner stub that runs the function immediately
  const schedule = {
    run: jest.fn(async ({ fn }: { fn: () => Promise<void> }) => {
      await fn();
    }),
  } as any;

  const provider = new MyEntityProvider({
    config: { apiUrl: 'http://x/items' },
    logger: mockServices.rootLogger().child({ pluginId: 'catalog' }),
    schedule,
  });

  // Stub global fetch (or extract fetchEntities and mock it directly)
  (global as any).fetch = jest.fn().mockResolvedValue({
    json: async () => [{ name: 'foo', owner: 'group:default/team-a' }],
  });

  await provider.connect(connection);

  expect(connection.applyMutation).toHaveBeenCalledWith({
    type: 'full',
    entities: [
      expect.objectContaining({
        entity: expect.objectContaining({
          metadata: expect.objectContaining({ name: 'foo' }),
        }),
      }),
    ],
  });
});
```

Notes:

- For complex providers, prefer extracting the fetch logic into a separate helper that takes a typed external client. Mock the client; don't mock global `fetch`.
- The scheduler runner is a small stub — there's no `mockServices.scheduler.factory()` shorthand for "run immediately on connect".

### Incremental provider

Test `next()` directly by stepping through pages — there's no framework loop to mock:

```ts
import { mockServices } from '@backstage/backend-test-utils';
import { MyIncrementalProvider } from './MyIncrementalProvider';

it('paginates through pages and signals done', async () => {
  const provider = new MyIncrementalProvider(
    { apiUrl: 'http://x', apiToken: 't', pageSize: 2 },
    mockServices.rootLogger().child({ pluginId: 'catalog' }),
  );

  // Hand-rolled context with a fake client whose listItems returns scripted pages:
  const calls: Array<{ pageToken?: string }> = [];
  const context = {
    client: {
      listItems: jest.fn(async ({ pageToken }: { pageToken?: string }) => {
        calls.push({ pageToken });
        if (!pageToken) {
          return {
            items: [{ name: 'a' }, { name: 'b' }],
            nextPageToken: 'page2',
          };
        }
        return { items: [{ name: 'c' }], nextPageToken: undefined };
      }),
      close: jest.fn(),
    },
  } as any;

  const first = await provider.next(context);
  expect(first.done).toBe(false);
  expect(first.entities).toHaveLength(2);
  if (!first.done) {
    expect(first.cursor).toEqual({ pageToken: 'page2' });
  }

  const second = await provider.next(context, { pageToken: 'page2' });
  expect(second.done).toBe(true);
  expect(second.entities).toHaveLength(1);
});

it('around opens and closes the client', async () => {
  const provider = new MyIncrementalProvider(
    { apiUrl: 'http://x', apiToken: 't', pageSize: 2 },
    mockServices.rootLogger().child({ pluginId: 'catalog' }),
  );

  const burst = jest.fn();
  await provider.around(burst);

  expect(burst).toHaveBeenCalled();
  // ...if your around() opens a real client, assert close was called.
});
```

Notes:

- Don't try to integration-test the full incremental loop — that involves the framework's database, scheduler, and event bus. Trust that the framework is correctly tested upstream; verify your provider's `next` and `around` in isolation.
- Cursor types must be JSON-serializable. If a unit test of `next` returns a cursor that JSON.parse(JSON.stringify(...)) round-trips lossy-ly, that's a bug.
