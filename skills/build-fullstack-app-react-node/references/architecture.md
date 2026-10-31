# Architecture Reference

The proven layout: **one Node process, one port, one origin**, serving both the API and the built frontend. This is simpler than running a separate Vite dev server behind a proxy — there's one thing to start in dev, one thing to deploy in prod, and the browser's relative `/api/...` calls work identically in both, so there's no CORS and no proxy config to keep in sync.

```
<app-name>/
  server/
    index.js         # loads env, mounts /api, then Vite middleware (dev) or dist/ (prod)
    db.js             # pg.Pool, transaction() helper, schema setup, idempotent seeding
    catalog.js         # seed data (products, or resources — rename per domain)
    schema.sql          # CREATE TABLE IF NOT EXISTS ... (idempotent, safe to re-run)
    api.js               # Express router: all /api/* routes + error handler
  src/
    api/
      shopApi.js           # fetch wrapper, one function per endpoint, ApiError class
      storage.js             # localStorage wrapper (only needed for guest/no-auth apps)
    shared/
      pricing.js              # totals/tax/shipping math — used by BOTH client and server
      validation.js             # checkout/booking field rules — used by BOTH
      status.js                   # derives display status from timestamps — used by BOTH
    context/
      ShopContext.jsx               # cart/selection state, wraps shared pricing
    components/
    pages/
      Home.jsx
      Detail.jsx                       # product or resource detail
      Checkout.jsx
      Orders.jsx / Bookings.jsx          # history + confirmation
  public/
  index.html
  vite.config.js
  package.json
  .env.example
  .gitignore
```

## `package.json` essentials

```json
{
  "type": "module",
  "private": true,
  "engines": { "node": ">=20.12" },
  "scripts": {
    "dev": "node server/index.js --dev",
    "build": "vite build",
    "lint": "oxlint",
    "start": "node server/index.js"
  },
  "dependencies": {
    "express": "^5.3.0",
    "pg": "^8.23.1",
    "react": "^19.2.8",
    "react-dom": "^19.2.8",
    "react-router-dom": "^7.18.4"
  },
  "devDependencies": {
    "@types/react": "^19.2.18",
    "@types/react-dom": "^19.2.7",
    "@vitejs/plugin-react": "^6.1.1",
    "oxlint": "^1.81.0",
    "vite": "^8.3.0"
  }
}
```

No `concurrently`, no `nodemon`, no dev-only proxy dependency — `server/index.js` itself decides whether to run Vite in middleware mode or serve the build, based on a `--dev` flag. Pin dependency versions you actually know are current; don't copy these verbatim without checking.

## `vite.config.js`

Nothing unusual needed — no `server.proxy` block, because Express mounts Vite's middleware directly in the same process in dev:

```js
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [react()],
})
```

## `server/index.js`

```js
// Single process serving both the API (/api/*) and the React app.
//   Development (`npm run dev`):  Vite runs inside Express as middleware, with hot reload.
//   Production  (`npm start`):    Express serves the pre-built files in dist/.
import path from 'node:path';

// Load .env for local development; on a host, set real environment variables instead.
try {
  process.loadEnvFile(path.resolve(import.meta.dirname, '..', '.env'));
} catch {
  // No .env file: rely on the environment.
}

const { default: express } = await import('express');
const { api } = await import('./api.js');
const { initDatabase } = await import('./db.js');

await initDatabase();

const isDev = process.argv.includes('--dev');
const PORT = Number(process.env.PORT) || 3000;
const root = path.resolve(import.meta.dirname, '..');

const app = express();
app.disable('x-powered-by');
app.use('/api', api);

if (isDev) {
  const { createServer } = await import('vite'); // dev-only dependency
  const vite = await createServer({ root, server: { middlewareMode: true }, appType: 'spa' });
  app.use(vite.middlewares);
} else {
  const dist = path.join(root, 'dist');
  // Vite's hashed bundles never change, so they can be cached forever.
  app.use('/assets', express.static(path.join(dist, 'assets'), { maxAge: '1y', immutable: true }));
  app.use(express.static(dist, { index: false }));
  // Client-side routes all get index.html. Requests for missing files (anything
  // with an extension) get a real 404 rather than HTML.
  app.use((req, res, next) => {
    if (req.method !== 'GET' || path.extname(req.path)) return next();
    res.sendFile(path.join(dist, 'index.html'), { maxAge: 0 });
  });
}

app.listen(PORT, () => {
  console.log(`${isDev ? '(dev) ' : ''}running at http://localhost:${PORT}`);
});
```

**Why `process.loadEnvFile` runs before the dynamic imports**: `db.js` reads `process.env.DATABASE_URL` at module-evaluation time (to build the `pg.Pool`), not inside a function. If `express`/`api.js`/`db.js` were imported with static `import` statements at the top of the file, they'd all evaluate before the `.env` file is loaded and `DATABASE_URL` would be undefined. Dynamic `import()` after `loadEnvFile` guarantees the ordering. Keep this pattern — don't "clean up" the dynamic imports into static ones.

**Why the SPA fallback checks `path.extname(req.path)`**: without it, a typo'd or stale asset URL (e.g. after a redeploy changes hashed filenames) silently returns `index.html` with a 200 instead of a 404, which then fails confusingly in the browser (a JS file that's actually HTML) instead of a clear network error.

## `.gitignore` must include

```
node_modules
dist
.env
*.log
```

Verify this before the first commit. Also check for any other file in the repo root that might hold a real credential someone pasted in while setting things up (a notes/scratch file, a requirements doc) — `.gitignore` only protects `.env` by name, not anything else.

## `.env.example`

```
# Neon console -> your project -> Connect -> copy the connection string.
# Neon shows ...?sslmode=require; change to sslmode=verify-full (same security, avoids a pg warning).
DATABASE_URL=postgresql://USER:PASSWORD@HOST.neon.tech/neondb?sslmode=verify-full

# Optional
# PORT=3000
```

If the app needs auth (Phase 0), add `BETTER_AUTH_SECRET`/`BETTER_AUTH_URL` here too — see `references/backend-gotchas.md`.

## `src/api/shopApi.js` (client API wrapper)

```js
export class ApiError extends Error {
  constructor(status, message, fields) {
    super(message);
    this.status = status;
    this.fields = fields;
  }
}

async function request(path, options = {}) {
  let res;
  try {
    res = await fetch('/api' + path, {
      ...options,
      headers: { 'Content-Type': 'application/json', ...options.headers },
      body: options.body && JSON.stringify(options.body),
    });
  } catch {
    throw new ApiError(0, 'Could not reach the server. Check your connection and try again.');
  }
  const data = await res.json().catch(() => ({}));
  if (!res.ok) throw new ApiError(res.status, data.error ?? `Request failed (${res.status})`, data.fields);
  return data;
}

// One function per endpoint, e.g.:
export const getProducts = () => request('/products');
export const placeOrder = (body) => request('/orders', { method: 'POST', body });
```

The `fields` map lets validation errors point at specific form inputs (see `references/backend-gotchas.md` on shared validation) — carry it through rather than collapsing every error to a single string message.
