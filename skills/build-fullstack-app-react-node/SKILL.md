---
name: build-fullstack-app-react-node
description: Use this skill whenever the user wants to scaffold, build, or deploy a full-stack web app for ANY domain using React + Vite on the frontend and Node.js + Express on the backend with Postgres (Neon) for persistence — an online store or catalog app, a booking/reservation app (tennis courts, doctor/dentist appointments, salon or spa slots, equipment or vehicle rentals, meeting rooms), or anything shaped like "browse a catalog, pick item(s) or a time slot, check out." Trigger on phrases like "build me an online store for X", "scaffold an app where people can buy/order/reserve/book Y", "deploy a React and Node app", "I need a shopping cart and checkout for Z", or "clone an e-commerce site for W" — even if the user never says "skill," "e-commerce," or "booking" explicitly. Also use it for requests to take an existing half-built React+Express+Postgres app the rest of the way to a live deployment. Make sure to use this skill rather than improvising the stack from scratch — it encodes hard-won lessons (and the exact failure modes) from actually building and deploying a React + Vite + Express + Postgres app this way, including the single-process dev/deploy pattern, atomic inventory decrements, and simulated order-status derivation.
---

# Build Full-Stack App (React + Node + Postgres)

Scaffolds and deploys a full-stack app — React + Vite frontend, Express API, Postgres (Neon) for persistence, deployed as a **single process on a single origin** (no separate frontend/backend to host, no CORS, no dev-time proxy). The architecture and every gotcha below come from actually building one (an e-commerce app: products, cart, checkout, inventory, order history) and hitting real failures along the way. Follow this rather than reinventing the architecture.

**Read the reference files as you reach each phase — don't front-load all of them.** Each is only needed at a specific point in the build.

## The two resource shapes this covers

Despite the different domains, every app this skill builds is "browse a catalog of X, select some of it, check out, see order/booking history." X takes one of two shapes, and the shape determines the schema and the concurrency-safety mechanism — get this right in Phase 0, because it's expensive to change later:

1. **Stock-count shape** — the catalog item is a countable thing (a product, a ticket type, a seat class) and checkout decrements a quantity. Two buyers racing for the last unit must not both succeed. *Example: an online store.*
2. **Time-range shape** — the catalog item is a resource booked for a start/end time, and two bookings on the same resource must never overlap. *Example: a tennis court, a salon chair, a rental van, a meeting room, a doctor's appointment slot.*

`references/domain-shapes.md` covers the schema and the correct concurrency mechanism for each — they are **not** interchangeable (a row lock that's sufficient for a stock count is not sufficient for preventing overlapping time ranges; don't reuse one domain's mechanism for the other).

## Phase 0 — Discovery (always do this first)

Don't start scaffolding from assumptions. Ask (via `AskUserQuestion` if the conversation doesn't already answer these):

1. **Domain and vocabulary** — what's being sold/booked, and what's the action called (buy/order/reserve/book)? This drives naming throughout (routes, table names, UI copy) — don't ship generic "item"/"resource" labels once a real domain is known.
2. **Resource shape** — stock-count or time-range (see above)? If time-range, also pin down the slot model: a fixed grid (hourly court blocks), variable-duration spans (multi-day rental), or provider-defined windows (a doctor's available times) — this shapes the booking UI as well as the schema.
3. **Auth** — does checkout/booking need a signed-in account, or is a demo/guest flow acceptable (cart and history live in `localStorage`, no login)? A guest-only app is meaningfully simpler to build and deploy (no auth plumbing, no session cookies, no "forgot password" flow) and is often the right call for a first pass or a demo — don't default to requiring auth just because it seems more "complete." If the user wants multi-visitor correctness (e.g. "everyone should only see their own orders") auth is required; if it's a single-demo-user or trusted-internal tool, it isn't.
4. **Pricing** — flat per-item, per-hour/day, or free/no real payment (this skill builds a demo checkout that collects card-shaped fields and keeps only the last 4 digits — it does not integrate a real payment processor unless the user explicitly asks for one, which is out of scope here).
5. **Deploy target** — confirm Render + Neon Postgres (the proven combination), and whether to reuse an existing Neon database or create a fresh one. If reusing one across multiple apps built with this skill, make sure table names won't collide.

Five minutes of questions here produces a materially better app than guessing — don't skip it under time pressure.

## Phase 1 — Scaffold

Read `references/architecture.md` for the full file layout and copy-paste templates: `server/index.js`, `server/db.js`, `src/api/`, `vite.config.js`, `package.json`, `.gitignore`, `.env.example`. It uses a **single Node process** for both dev and prod — Vite runs in middleware mode inside Express rather than as a second process behind a proxy, so there's one thing to start, one thing to deploy, and the browser always calls relative `/api/...` URLs. Rename placeholder vocabulary (`products`/`orders`, or `resources`/`bookings`) to the domain's real words from Phase 0 as you go.

Set `"engines": { "node": ">=20.12" }` in `package.json` (Render needs it to pick a modern Node version) and add `oxlint` for linting (`npm run lint`) — both are part of the proven template.

## Phase 2 — Domain core (the shape-specific part)

Read `references/domain-shapes.md` before writing `schema.sql` or the checkout/booking route. It covers both schemas, the authoritative-pricing rule (never trust client-sent prices — recompute server-side from the database inside the transaction, for both shapes), and the correct concurrency mechanism for whichever shape Phase 0 picked.

## Phase 3 — Backend plumbing and gotchas

Read `references/backend-gotchas.md`. Every item is a real failure that was hit and fixed, not a hypothetical best practice — treat it as a pre-flight checklist, not background reading. It covers: environment loading, the Postgres connection pool, `NUMERIC`-as-string, idempotent seeding, Express 5's async error handling, the shared-code-between-client-and-server trap, simulated status derivation (useful for both shapes — "Processing → Shipped → Delivered" and "Confirmed → Completed" are the same pattern), and — only if Phase 0 said auth is required — the Better Auth setup and its own specific pitfalls.

## Phase 4 — Verify locally before touching deployment

`npm install && npm run dev`, then actually exercise it end to end: browse the catalog, add to cart / pick a slot, complete checkout, **try to oversell/double-book on purpose and confirm the server rejects it** (open two tabs and race them if needed — this is the one thing that must not be taken on faith), check order/booking history, cancel something if that's supported. Run `npm run lint`. Don't move to deployment on an unverified app.

## Phase 5 — Git and GitHub (needs the user's hands)

This cannot be fully automated — there are no stored GitHub credentials in a fresh environment, and pushing requires an interactive browser OAuth popup that a sandboxed tool session cannot complete. Concretely:

1. Do the local, reversible part yourself: `git init`, verify `.gitignore` excludes `.env`/`node_modules`/`dist`, stage, commit. **Also check for any other file that might carry a real credential** — a notes file, a scratch doc, a pasted connection string saved outside `.env` — `.gitignore` only protects `.env` itself.
2. Ask the user for the GitHub repo URL (they create an empty repo at github.com/new) — never invent or guess one.
3. Give them the exact push command to run themselves, prefixed with `!` so it runs in their real interactive terminal: `! cd <path>; git push -u origin main`.
4. **Verify the push actually landed**: compare `git ls-remote origin`'s HEAD SHA against local `git log --oneline -1`. A remote commit message like "Add files via upload" means they pushed via GitHub's web UI instead — re-sync before proceeding to deploy.

## Phase 6 — Render deploy

Read `references/deployment-render.md` for the full playbook: the `render.yaml` Blueprint template, the dashboard flow, the faster path when the user hands you a Render API key, and the exact env-var-missing crash signature this stack produces (recognize it immediately rather than re-diagnosing from scratch).

## Phase 7 — Write the app's CLAUDE.md

Once deployed, write a `CLAUDE.md` for the new repo documenting: the domain-specific schema and which resource shape it uses, the concurrency-safety mechanism and why, whether auth is present, where client state vs. server state lives, and the deployment target. This is what makes a future session on this specific app productive on the first message instead of re-deriving the architecture — don't skip it just because the app is "done."
