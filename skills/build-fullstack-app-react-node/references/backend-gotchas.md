# Backend & Database Gotchas

Each item here is a failure that was actually hit while building and deploying this stack, not a hypothetical best practice. Treat this as a pre-flight checklist before writing `server/index.js`, `schema.sql`, and the routes.

## 1. `pg.Pool` needs an `error` listener

```js
pool.on('error', (err) => {
  console.error('Unexpected error on idle Postgres client:', err.message);
});
```

Neon closes idle connections from its side. Node's default behavior for an unhandled `error` event on an `EventEmitter` — which is what an idle pool client fires when its connection is cut — is to **crash the process**. Without this listener, the app dies randomly under low traffic with nothing informative in the logs unless you know to check for this.

## 2. `NUMERIC` columns come back as strings

`pg` returns Postgres `NUMERIC`/`DECIMAL` as JS strings, to avoid float precision loss. Convert explicitly at the API boundary, every time:

```js
price: Number(row.price),
rating: Number(row.rating),
```

Forgetting this doesn't error — it silently sends `"49.99"` as a string, which then breaks arithmetic (`total + shipping` becomes string concatenation) in ways that are easy to miss in casual testing but show up as wrong totals. Write one `toProduct`/`toOrder` mapper per entity that does all the `Number()` conversions in one place, rather than converting ad hoc in each route.

## 3. Idempotent schema + seeding, run on every startup

```js
export async function initDatabase() {
  await pool.query(fs.readFileSync(path.join(import.meta.dirname, 'schema.sql'), 'utf8'));
  // ... build a multi-row INSERT from the seed data ...
  const { rowCount } = await pool.query(
    `INSERT INTO products (...) VALUES (...), (...) ON CONFLICT (id) DO NOTHING`,
    values,
  );
  if (rowCount) console.log(`Seeded ${rowCount} row(s).`);
}
```

`schema.sql` uses `CREATE TABLE IF NOT EXISTS` throughout, and seeding uses `ON CONFLICT DO NOTHING` — both are safe to run every time the server starts, including on every redeploy. This also means **editing a seed row's price/stock in the seed file does not update an already-seeded database** — existing rows are left alone on purpose, so real stock/booking data survives redeploys. If the user wants to change a live value, that's an `UPDATE` or a migration, not a seed-file edit.

## 4. Express 5's async error handling

Express 5 forwards a rejected promise from an async route handler straight to the error-handling middleware — you don't need `try/catch` + `next(err)` in every route. Throw a typed error and let it propagate:

```js
class HttpError extends Error {
  constructor(status, message, fields) {
    super(message);
    this.status = status;
    this.fields = fields; // optional: { fieldName: "message" } for form validation
  }
}

// in a route:
if (rows.length === 0) throw new HttpError(404, 'Not found');

// one error handler at the bottom of the router, after all routes:
api.use((err, req, res, next) => {
  if (err instanceof HttpError) return res.status(err.status).json({ error: err.message, fields: err.fields });
  if (err.type === 'entity.parse.failed') return res.status(400).json({ error: 'Invalid JSON body' });
  console.error(err);
  res.status(500).json({ error: 'Something went wrong. Please try again.' });
});
```

Confirm the Express major version actually in use before relying on this — if a project ends up on Express 4 instead, async throws need explicit `next(err)` or an async-wrapper helper, because Express 4 does not catch them automatically.

## 5. Shared code between the browser and the server

Pricing math, checkout/booking validation, and status derivation belong in one file each under `src/shared/`, imported by *both* the React app and `server/api.js` (via a relative path like `../src/shared/pricing.js`). The browser uses them for instant UI feedback; the server uses the exact same logic for enforcement. **Change a rule in one place, not two** — this is the main defense against the client and server silently disagreeing about a total or a validation rule.

The catch: Node runs these files directly, with no bundler, while the React side goes through Vite. That means inside `src/shared/`:
- Use explicit `.js` extensions on any imports between shared files (`import { x } from './pricing.js'`, not `'./pricing'`) — Node's ESM loader requires it; Vite would have resolved the extension-less form for you, masking the problem until the server tries to boot.
- Don't use browser-only globals (`window`, `localStorage`) or JSX in these files — they need to run unmodified under plain Node.

React components importing from `src/shared/` can omit the extension, since Vite resolves it — the constraint is only on imports *within* `src/shared/` and on the server's imports of it.

## 6. Loading environment variables

Use `process.loadEnvFile(pathToEnvFile)` (built into modern Node) rather than adding a `dotenv` dependency, and call it before any module that reads `process.env` at import time is loaded — see the dynamic-import ordering in `references/architecture.md`. Wrap it in `try/catch`: on a real host, there is no `.env` file and real environment variables are set directly, so a missing file should be silently fine, not a crash.

## 7. Windows dev-server port cleanup

On Windows, stopping `npm run dev` (e.g. with Ctrl+C in a terminal that launched it via `npm`) can leave the child `node` process holding the port, because `npm` doesn't always forward the signal to its child. If a later `npm run dev` fails with "port already in use," either run `node server/index.js --dev` directly (so Ctrl+C stops the actual process) or find and kill the leftover process with `netstat -ano` / Task Manager. Mention this to the user if they report a stuck dev server rather than guessing at other causes first.

## 8. If Phase 0 said auth is required

Use Better Auth (`better-auth` + `@better-auth/cli`), and treat these as the checklist — same failure modes regardless of domain:

- Mount Better Auth's handler **before** `express.json()`: `app.all('/api/auth/*splat', toNodeHandler(auth)); app.use(express.json());`. Better Auth reads the raw request body itself; if `express.json()` runs first it consumes the stream and auth requests silently get an empty body.
- `@better-auth/cli`'s generated schema can lag the `better-auth` core package (a runtime error like `column "issuer" does not exist` the first time that code path runs). Patch it additively in `schema.sql`: `ALTER TABLE account ADD COLUMN IF NOT EXISTS issuer TEXT;` — don't fight it with a full migration tool.
- Run the auth migration **before** the app-schema seed on a fresh database — the app's own tables have a foreign key to Better Auth's `user` table.
- Gate protected pages by reading the session directly at the call site, not through a wrapper/HOC:
  ```jsx
  const { data: session, isPending } = authClient.useSession();
  if (isPending) return <p>Loading…</p>;
  if (!session) return <Navigate to="/login" state={{ returnTo: location.pathname }} replace />;
  ```
  `Login.jsx` reads `location.state?.returnTo` and navigates there after sign-in. This keeps each page's auth requirement visible at the call site — there are usually only one or two protected pages (checkout/booking, history), so the duplication cost is low and an abstraction isn't worth it yet.

If Phase 0 said auth is **not** required, don't add any of the above — cart/wishlist/selection state lives in `localStorage` via a tiny wrapper (see `src/api/storage.js` in `references/architecture.md`), and routes like `GET /orders` simply return everyone's data, which is fine for a guest-only demo but should be called out explicitly as a known limitation in the app's own `CLAUDE.md` (Phase 7) so a future session doesn't mistake it for an oversight.
