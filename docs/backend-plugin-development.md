# Creating a Backend API Plugin

This guide covers building a backend API plugin using this repo's new backend system.

## Overview

This repo uses Backstage's **new backend system** (`createBackend()` + `backend.add()`). Plugins are self-contained modules that declare their own routes, services, and database needs.

---

## Step 1 — Scaffold the plugin

From the repo root:
```bash
yarn new
```
Select **"backend-plugin"** when prompted. Give it a name, e.g. `my-api`. This creates `plugins/my-api-backend/`.

The scaffold generates:
```
plugins/my-api-backend/
├── src/
│   ├── index.ts          # Public exports
│   ├── plugin.ts         # Plugin definition
│   └── router.ts         # Express router (HTTP routes)
├── package.json
└── README.md
```

---

## Step 2 — Understand the plugin definition

The generated `plugin.ts` uses `createBackendPlugin`:

```ts
import { createBackendPlugin } from '@backstage/backend-plugin-api';
import { createRouter } from './router';

export const myApiPlugin = createBackendPlugin({
  pluginId: 'my-api',
  register(env) {
    env.registerInit({
      deps: {
        httpRouter: coreServices.httpRouter,
        logger: coreServices.logger,
        database: coreServices.database,   // optional — only if you need a DB
        config: coreServices.rootConfig,   // optional — only if you need config
      },
      async init({ httpRouter, logger }) {
        httpRouter.use(await createRouter({ logger }));
      },
    });
  },
});
```

**Key concepts:**
- `pluginId` — becomes the URL prefix: `/api/my-api/...`
- `deps` — services injected automatically by the backend framework
- `httpRouter` — mounts your Express router at the plugin's URL prefix
- Each plugin gets its **own isolated database schema** if it requests `coreServices.database`

---

## Step 3 — Define your routes

In `router.ts`:

```ts
import { Router } from 'express';
import { LoggerService } from '@backstage/backend-plugin-api';

export async function createRouter(options: {
  logger: LoggerService;
}): Promise<Router> {
  const { logger } = options;
  const router = Router();

  router.get('/health', (_, res) => {
    res.json({ status: 'ok' });
  });

  router.get('/items', async (req, res) => {
    logger.info('Fetching items');
    res.json({ items: [] });
  });

  return router;
}
```

Your routes will be available at:
- `GET /api/my-api/health`
- `GET /api/my-api/items`

---

## Step 4 — Register the plugin in the backend

In `packages/backend/src/index.ts`, add one line:

```ts
backend.add(import('@internal/backstage-plugin-my-api-backend'));
```

---

## Step 5 — Available core services

These are injected via `deps` in your plugin definition:

| Service | Key | What it gives you |
|---|---|---|
| HTTP router | `coreServices.httpRouter` | Mount Express routes |
| Logger | `coreServices.logger` | Structured logging |
| Database | `coreServices.database` | Knex client, isolated schema |
| Config | `coreServices.rootConfig` | Read `app-config.yaml` values |
| Auth | `coreServices.auth` | Issue/verify service tokens |
| HTTP auth | `coreServices.httpAuth` | Authenticate incoming requests |
| Scheduler | `coreServices.scheduler` | Schedule recurring tasks |
| Cache | `coreServices.cache` | Key/value cache |

---

## Step 6 — Using the database

If your plugin needs to persist data:

```ts
deps: {
  database: coreServices.database,
},
async init({ database }) {
  const db = await database.getClient();
  // db is a Knex instance scoped to your plugin's schema
  await db.schema.createTableIfNotExists('items', table => {
    table.increments('id');
    table.string('name').notNullable();
    table.timestamps(true, true);
  });
}
```

In dev this uses SQLite, in production PostgreSQL — no config changes needed in your plugin.

---

## Step 7 — Authenticating requests

To protect routes so only logged-in users can call them:

```ts
deps: {
  httpRouter: coreServices.httpRouter,
  httpAuth: coreServices.httpAuth,
},
async init({ httpRouter, httpAuth }) {
  const router = Router();

  router.get('/protected', async (req, res) => {
    // Throws if request has no valid credentials
    await httpAuth.credentials(req, { allow: ['user'] });
    res.json({ secret: 'data' });
  });

  httpRouter.use(router);
  httpRouter.addAuthPolicy({ path: '/health', allow: 'unauthenticated' });
}
```

---

## Step 8 — Run it locally

```bash
yarn start
```

The backend starts at `http://localhost:7007`. Your plugin routes are at `http://localhost:7007/api/my-api/...`.

---

## Key things to know about this repo

- **`plugins/` is empty** — all custom plugins go here
- **New backend system only** — don't use the old `createRouter` + `PluginEnvironment` pattern from older Backstage docs; it won't work here
- **After any config change** in `app-config.production.yaml`, you must rebuild and push the Docker image — configs are baked in at build time
- **Plugin URL prefix** is always `/api/<pluginId>/`
