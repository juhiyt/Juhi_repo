# Deployment Reference — GitHub + Render

## The core constraint: two steps need the user's hands

A sandboxed tool session has no stored GitHub or Render credentials, and both platforms gate the relevant action behind an interactive browser login (GitHub via Git Credential Manager's OAuth popup on `git push`; Render via its dashboard login). Don't spend retries trying to work around this — confirm the failure mode once, then hand it to the user with an exact command. The failure (`fatal: Cannot prompt because user interactivity has been disabled` / `terminal prompts disabled`) is deterministic in a non-interactive session; repeatedly re-running `git push` just burns turns.

## `render.yaml` (Blueprint)

```yaml
services:
  - type: web
    name: <app-name>
    runtime: node
    plan: free
    buildCommand: npm install && npm run build
    startCommand: npm start
    envVars:
      - key: DATABASE_URL
        sync: false
```

If Phase 0 required auth, add:

```yaml
      - key: BETTER_AUTH_SECRET
        generateValue: true
      - key: BETTER_AUTH_URL
        sync: false
```

`sync: false` means Render prompts the user for that value in the dashboard instead of it living in the repo. `generateValue: true` auto-generates a secret. If the app exposes a lightweight `GET /api/health` route, add `healthCheckPath: /api/health` too — it's not required by this architecture (there's no separate frontend/backend health split to worry about), but it gives Render a cheap way to confirm the process is actually serving traffic, not just running. Free plan spins down after 15 minutes idle (cold start ~30–50s on the next request) — mention this and offer `plan: starter` if the user wants to avoid it.

## Git setup (do this part yourself — local and reversible)

1. Check for git; install via `winget install --id Git.Git -e` (Windows) if missing — a PATH refresh is needed in the same shell session, so call it via its full install path (e.g. `C:\Program Files\Git\cmd\git.exe`) or prepend `$env:Path += ";C:\Program Files\Git\cmd"` for the rest of the session.
2. `git init`, set `user.name`/`user.email` if unset, verify `.gitignore` excludes `.env`/`node_modules`/`dist` (check `git status`/`git add -A` output — `.env` must never appear staged), commit.
3. Rename branch to `main` if needed.

## Handing off the push

Ask the user for the GitHub repo URL (they create an empty repo at github.com/new — no README/gitignore/license, since the local repo already has those). Never guess a repo name or assume one exists.

```
git remote add origin <url>
```

Then give them this to run themselves, with the `!` prefix so it executes in their real interactive terminal rather than the sandboxed tool session:

```
! cd <path>; git push -u origin main
```

**Verify before trusting "it's done."** Check `git ls-remote origin` and compare its `HEAD` SHA against local `git log --oneline -1`. If they differ, or a fetched commit message reads like "Add files via upload" instead of your commit message, the user pushed through GitHub's web UI (drag-and-drop) instead of `git push` — the repo's content may not match what was scaffolded locally, which produces confusing deploy failures (wrong build/start commands detected, missing `render.yaml` fields) that look unrelated to the actual cause. Re-sync before debugging anything downstream.

## Render dashboard flow (needs the user's login)

1. dashboard.render.com → **New +** → **Blueprint** → connect GitHub → select the repo. Render detects `render.yaml`.
2. It prompts for the `sync: false` values: `DATABASE_URL` (their Neon connection string), and `BETTER_AUTH_URL` if auth is present (the assigned `https://<name>.onrender.com` URL — Render shows this during setup).
3. Apply/Deploy. If the final assigned URL differs from what was entered for `BETTER_AUTH_URL`, fix it in the service's Environment tab afterward and redeploy — see the crash signature below.

## Faster path: if the user provides a Render API key

Handle it with two non-negotiable rules, stated up front to the user:
1. **Never write the key to a file** — especially not inside the repo, even temporarily. Use it inline in shell commands / env vars for the session only.
2. **Tell the user to rotate it once you're done** (Render dashboard → Account Settings → API Keys) — it's now sitting in plaintext in the chat transcript, which is a real exposure even in a private session.

API calls (base `https://api.render.com/v1`, header `Authorization: Bearer <key>`):

```
GET  /services?name=<app-name>                          # find the service, get its id
POST /services/{id}/deploys                              # trigger a deploy, body {}
GET  /services/{id}/deploys/{deployId}                   # poll status: build_in_progress -> update_in_progress -> live | update_failed
PUT  /services/{id}/env-vars/{key}   body {"value": "…"} # set one env var without touching the others
GET  /logs?ownerId=<teamId>&resource=<serviceId>&direction=backward&limit=100   # runtime/build logs (ownerId comes from GET /services/{id})
```

Poll the deploy status every ~15s rather than assuming success — a build can succeed while the process still crashes on boot (`status: update_failed` after `build_successful`), which only shows up in the logs endpoint, not the deploy status alone.

**Do not attempt `GET /services/{id}/env-vars`** (reading current values) if that call gets blocked by an auto-mode/permission classifier — that's a deliberate guard against pulling secret values into the session's context. Set values with the `PUT` endpoint above (write-only) and ask the user to confirm values visually in the dashboard if you need to verify something without reading it back.

## The exact crash to recognize immediately

If a deploy reaches a successful build but then `update_failed`, or the logs show the server logging "listening on..." followed immediately by a crash, check for this signature:

```
DATABASE_URL is not set. ...
```

or, if auth is present:

```
[BetterAuthError: You are using the default secret. Please set `BETTER_AUTH_SECRET` ...]
```

This means the relevant env var was never actually set on the service — commonly because the Blueprint's `sync: false` prompts never fired (e.g. the service was created by hand instead of via Blueprint, or content was uploaded manually rather than pushed — see the git verification note above). Fix: pull `DATABASE_URL` from the user's local `.env` (read-only, don't echo it unnecessarily) and set it (plus `BETTER_AUTH_SECRET`/`BETTER_AUTH_URL` if applicable) via the `PUT env-vars` call.

**Env var changes do not reliably trigger an automatic redeploy** — after setting them via API or dashboard, explicitly `POST /services/{id}/deploys` (or click Manual Deploy) and poll it to `live`. Then verify with an actual request, not just the status field:

```
GET https://<app>.onrender.com/api/products     # (or /api/resources) — expect real seeded data, confirms DB connectivity too
```
