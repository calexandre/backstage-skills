# Pattern: createTemplateAction

The action is defined by a single `createTemplateAction` call. Schemas declare the shape of `input` and `output`; the handler is where the work happens.

## Standalone usage

### Basic action

```ts
import { createTemplateAction } from '@backstage/plugin-scaffolder-node';

export const myAction = createTemplateAction({
  id: 'acme:user:create',
  description: 'Creates a user record in the acme directory',
  schema: {
    input: {
      name: z => z.string().describe('Full name of the user'),
      email: z => z.string().email().describe('Email address'),
      groups: z => z.array(z.string()).optional().describe('Group memberships'),
    },
    output: {
      userId: z => z.string().describe('The id of the created user'),
    },
  },
  async handler(ctx) {
    ctx.logger.info(`Creating user ${ctx.input.name}`);

    const userId = await createUserInExternalSystem({
      name: ctx.input.name,
      email: ctx.input.email,
      groups: ctx.input.groups ?? [],
    });

    ctx.output('userId', userId);
  },
});
```

Key points:

- `id` uses `:` as a hierarchy separator. Stable, namespaced, lowercase recommended.
- `schema.input` and `schema.output` use the **factory function form**: each field is `field: z => z.<type>(...)`. The framework calls these factories with its pinned Zod instance.
- `ctx.input` is fully typed from the input schema — `ctx.input.name` is `string`, `ctx.input.groups` is `string[] | undefined`.
- `ctx.output(name, value)` is the only way to emit outputs. Outputs are referenced in subsequent template steps as `${{ steps.this-step.output.userId }}`.
- The handler is `async` and returns `void`.

### Working in the workspace

The scaffolder runs each template step in a shared `workspacePath`. Actions read and write files there; later steps see those files:

```ts
import * as fs from 'fs/promises';
import * as path from 'path';

export const writeReadme = createTemplateAction({
  id: 'acme:files:write-readme',
  schema: {
    input: {
      title: z => z.string(),
      description: z => z.string(),
    },
  },
  async handler(ctx) {
    const targetPath = path.join(ctx.workspacePath, 'README.md');
    await fs.writeFile(
      targetPath,
      `# ${ctx.input.title}\n\n${ctx.input.description}\n`,
      'utf-8',
    );
    ctx.logger.info(`wrote ${targetPath}`);
  },
});
```

### Dry-run support

For actions where the user can preview a template before publishing, opt into dry-run mode:

```ts
export const callExternal = createTemplateAction({
  id: 'acme:external:call',
  supportsDryRun: true,
  schema: {
    input: { url: z => z.string() },
    output: { status: z => z.number() },
  },
  async handler(ctx) {
    if (ctx.isDryRun) {
      ctx.logger.info(`[dry-run] would POST to ${ctx.input.url}`);
      ctx.output('status', 200);
      return;
    }

    const response = await fetch(ctx.input.url, { method: 'POST' });
    ctx.output('status', response.status);
  },
});
```

### Acting as the user

When the action needs to do something on behalf of the user who triggered the template (e.g. open a PR as them), use `ctx.getInitiatorCredentials()`:

```ts
async handler(ctx) {
  const credentials = await ctx.getInitiatorCredentials();
  // Pass credentials to coreServices.auth.getPluginRequestToken({ onBehalfOf: credentials, ... })
  // — but actions can't depend on coreServices directly; receive the auth service
  // via the module's deps and pass it into the action factory if needed.
}
```

For actions that need core services, parameterize the action's factory function and inject services from the module's `init`. See `references/module-registration.md`.

## Post-scaffold usage

When invoked right after `creating-backstage-plugin` scaffolds a `scaffolder-backend-module`:

1. Create `plugins/<scaffolded-id>/src/actions/<actionName>.ts` with the `createTemplateAction` call.
2. Pick an action `id` from the scaffolded plugin id (e.g. plugin `scaffolder-backend-module-acme-slack` → action id `acme:slack:notify`).
3. Define a minimal input schema (at least one field) and emit at least one output to demonstrate the wiring.
4. Add `@backstage/plugin-scaffolder-node` to `dependencies`.

## Tests

The action is a plain object — call its `handler` with a stubbed context:

```ts
import { mockServices } from '@backstage/backend-test-utils';
import * as path from 'path';
import * as fs from 'fs/promises';
import * as os from 'os';
import { writeReadme } from './writeReadme';

it('writes README.md with the supplied title and description', async () => {
  const workspacePath = await fs.mkdtemp(path.join(os.tmpdir(), 'scaf-'));

  await writeReadme.handler({
    logger: mockServices.rootLogger().child({}),
    workspacePath,
    input: { title: 'Hello', description: 'World' },
    output: jest.fn(),
    checkpoint: jest.fn(),
    createTemporaryDirectory: jest.fn(),
    getInitiatorCredentials: jest.fn(),
    task: { id: 'test-task' },
    isDryRun: false,
  } as any);

  const contents = await fs.readFile(
    path.join(workspacePath, 'README.md'),
    'utf-8',
  );
  expect(contents).toContain('# Hello');
  expect(contents).toContain('World');
});

it('emits userId output', async () => {
  const output = jest.fn();
  await myAction.handler({
    logger: mockServices.rootLogger().child({}),
    workspacePath: '/tmp/x',
    input: { name: 'Jane', email: 'jane@example.com' },
    output,
    checkpoint: jest.fn(),
    createTemporaryDirectory: jest.fn(),
    getInitiatorCredentials: jest.fn(),
    task: { id: 'test-task' },
  } as any);

  expect(output).toHaveBeenCalledWith('userId', expect.any(String));
});
```

Notes:

- The full `ActionContext` shape includes `task`, `signal`, `templateInfo`, `user`, etc. — for unit tests, `as any` after a partial cast is fine; in production code these are populated by the scaffolder.
- For dry-run logic, set `isDryRun: true` in the test ctx and assert that side effects don't happen.
