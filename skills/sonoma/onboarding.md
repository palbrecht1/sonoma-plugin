# Sonoma onboarding (first run in a repo)

Follow this the first time a repo uses Sonoma. The goal is to get the local
setup healthy, decide whether the repo needs a `sonoma.yaml` at all, author one
only if it does, and then offer to spin up the env.

Most repos with a `docker-compose.yml` need no `sonoma.yaml`. `sonoma up` infers
the contract from the ports the compose services publish. Author a contract only
when there is no compose file, or when you want to override what Sonoma infers.

The CLI is the `sonoma` command (it runs on the Bun runtime). Do the steps in
order. Do not skip ahead to `up` before `doctor` is green.

## 1. Make sure the CLI is installed

Check whether `sonoma` is on PATH:

```bash
command -v sonoma
```

If it is missing, ensure bun is present, then install the CLI globally:

```bash
curl -fsSL https://bun.sh/install | bash   # only if bun is missing
bun add -g @sonoma.sh/cli
```

## 2. Run the doctor

```bash
sonoma doctor
```

It reports four things: local deps (`bun`, `ssh`, `scp`, `mutagen`), auth state,
and whether the control plane is reachable. Read the output before acting.

## 3. Get the doctor green

Fix each failing check, then re-run `doctor`. Repeat until every line is `ok`.

- **Missing `bun`:** install it, then retry.
  ```bash
  curl -fsSL https://bun.sh/install | bash
  ```
- **Missing `mutagen`:** install per platform. On macOS:
  ```bash
  brew install mutagen-io/mutagen/mutagen
  ```
  Otherwise download a release from https://github.com/mutagen-io/mutagen/releases
  and put it on `PATH`.
- **Missing `ssh` / `scp`:** these ship with the OS on macOS and most Linux.
  Surface it to the user with the platform-specific fix rather than guessing.
- **Not logged in:** run the browser login (use `--headless` for a device code
  when there is no browser, for example over SSH):
  ```bash
  sonoma login
  ```
- **Control plane unreachable:** this is not a local-setup problem you can fix.
  Stop here and tell the user. It is usually a network issue or a misconfigured
  server URL, not something `up` will recover from.

Do not move on until `doctor` passes.

## 4. Decide whether the repo needs a `sonoma.yaml`

Look for a compose file at the repo root (`docker-compose.yml`,
`docker-compose.yaml`, `compose.yml`, or `compose.yaml`).

**If one exists and the project is straightforward** (its services publish the
ports they serve on), you do not need to author anything. `sonoma up` infers the
contract:

- Every service that publishes a port gets a preview URL.
- The primary (the share link and the readiness target) is the service on a
  common web port (3000, 5173, 8080, 8000, 80, 443, ...), or the first published
  service when none matches.
- Seeding defaults to a no-op, and readiness defaults to waiting for the primary
  port to accept a connection.

`sonoma up` prints what it inferred before it spawns, so you confirm the exposed
services and the primary there. When that matches the project, skip authoring and
go to the final step.

**Author a `sonoma.yaml` (next step) only when:**

- the repo has no compose file, or
- the project needs a real seed command, a specific readiness probe, a particular
  primary service, or extra sync excludes, or
- a browser frontend calls a separate backend (you also wire the compose
  `environment:` block for cross-service URLs and CORS, covered below).

If none of those apply, you are done. Otherwise continue.

## 5. Author the project contract (`sonoma.yaml`), when needed

The CLI does not write this for you. When the decision above calls for a
contract, detect the project's shape, propose one, confirm it with the user field
by field, then write the file.

### Detect

Look at what the repo already declares:

- `docker-compose.yml` (or `compose.yaml`): the services and the ports they
  publish. This is the stack Sonoma will run inside the env.
- `.devcontainer/devcontainer.json`: an alternative build/run definition.
- `package.json` scripts (or the equivalent for the stack): a likely `seed`
  command, for example `db:reset`, `prisma migrate reset`, `seed`.

### Propose

From that, draft a `sonoma.yaml`:

- `environment.compose`: path to the compose file (relative to repo root).
- `expose`: one entry per service that needs a **public URL** (the browser reaches it). Mark the
  user-facing one `primary: true` (the share link + readiness target). Do NOT expose internal
  services (workers, an API only the server calls, datastores): they already reach each other by
  compose name inside the env, so exposing them only puts them on the public internet needlessly.
- `seed`: how to seed data, if the project needs it.
- `ready`: a single probe against the primary service, either an HTTP check or
  an arbitrary command, that the control plane retries until it passes.

### Confirm, then write

Show the user the proposed `sonoma.yaml` and confirm each field (especially the
primary service and port, and the `ready` probe). Do not write the file until
they agree. Then write it at the repo root.

### Schema reference

```yaml
# sonoma.yaml, lives in the repo root, authored once
version: 1                                # optional; omit for the current version
environment:
  compose: docker-compose.yml             # the stack, run inside the env
  # escape hatch when there is no compose: start: ./scripts/start.sh
seed: pnpm db:reset                       # optional: how to seed the env
preview:
  access: public                          # optional env-wide default; per-service access overrides it
expose:
  - service: web                          # must match a service in the compose
    port: 3000
    primary: true                         # the share link + readiness target
  - service: api
    port: 4000
    access: tenant-only                   # optional per-service; here api requires owner login, web stays public
ready: "curl -fsS http://localhost:3000/healthz"   # HTTP or arbitrary command
sync:                                     # optional: extra paths to keep out of the sync
  ignore:
    - dist/
    - .venv/
```

Rules that matter:

- Keep the surface tiny. Everything optional stays out unless the project needs
  it. The only hard requirements are how to bring the stack up
  (`environment.compose` or `environment.start`) and one `primary` exposed
  service.
- **Secrets never go in `sonoma.yaml`.** Config and secret values live in a
  gitignored `.env.sonoma`, synced into the env and loaded by the project's own
  compose via `env_file:`. The contract does not reference secrets at all.
- **Access defaults to `public`.** Set `access: tenant-only` (per service, or as an env-wide
  `preview.access` default) only for a service whose preview runs against real data the owner does
  not want anyone with the URL to see; it then requires the owner to log in before that service
  loads. A service's own `access` overrides the env default. Keep in mind `tenant-only` uses a browser
  session, so it blocks non-browser inbound (webhooks, external callbacks) to that service; use
  `public` for those.

### Forwarding shell env vars inline (`--env`)

If you would rather not keep a `.env.sonoma` on disk, you can forward named vars
straight from the shell you run `sonoma up` in:

```bash
export DATABASE_URL=postgres://...
export STRIPE_KEY=sk_live_...
sonoma up --env DATABASE_URL --env STRIPE_KEY   # repeat --env per var, by NAME
```

Each `--env NAME` reads that var's *value* from your current shell (so secret
values never appear in the command line or your history) and sends it with the
spawn. **No project changes are needed** -- you do not add anything to
`sonoma.yaml` or your compose file. Under the hood Sonoma injects the vars into
your services for you (for a compose project it merges a generated overlay into
`docker compose up`; for a `start:` command it sets them on the process env).

Notes:

- The vars are injected into every service, so any service can read them (like a
  root-level `.env` would). Values needing `${SONOMA_*}` still belong in the
  compose `environment:` block.
- Forwarding happens at *spawn*. Changing a `--env` value on a later `sonoma up`
  that reuses a live env has no effect; run `sonoma down` first to re-forward.
- `sonoma up` fails fast if a named var is unset in your shell, is not a valid env
  var name, or holds a multi-line value (keep multi-line secrets in `.env.sonoma`).
- `.env.sonoma` and `--env` compose freely; use either or both.

### Shareable URLs: the slug, cross-service APIs, and CORS

This is the part that makes the preview, and any API the frontend calls,
reachable from a browser. Get it right or the preview loads but its API calls
fail. It lives in your `docker-compose.yml`, so it applies whether or not you
author a `sonoma.yaml`: even an inferred contract needs these compose
`environment:` entries when a browser frontend calls a separate backend.

How addressing works:

- Each env gets a short id. Sonoma sets two environment variables on the VM
  where your compose runs, and these are the only handoff:
  - `SONOMA_SLUG`: the env's short id.
  - `SONOMA_BASE_DOMAIN`: `sonoma.sh`.
- Every service you list in `expose` is reachable at a single-level subdomain
  `https://${SONOMA_SLUG}-<service>-${SONOMA_HASH}.${SONOMA_BASE_DOMAIN}` (one
  wildcard cert covers all of them). The `primary` entry is the shareable link.

What this means for `expose`: add an entry for every service that is hit from
outside the env, not just the frontend. A browser-side frontend that calls a
separate backend means **both** `web` and `api` need to be exposed, because the
browser reaches the API over its public `<slug>-api-<hash>` URL, not `localhost`.

The interpolation rule that trips people up:

- `.env.sonoma` is loaded via compose `env_file:`, and those values are
  **literal**, so `${SONOMA_SLUG}` does NOT expand there. Put only static or
  secret values in `.env.sonoma`.
- Anything that must reference the slug/hash (the API URL the frontend points at,
  and the CORS allow-origin the API honors) goes in the compose `environment:`
  block, which compose **does** interpolate, because Sonoma set those vars on the
  VM.

So in the project's `docker-compose.yml`:

```yaml
services:
  web:
    environment:
      # the frontend (in the browser) calls the API at its public URL, not localhost
      - NEXT_PUBLIC_API_URL=https://${SONOMA_SLUG}-api-${SONOMA_HASH}.${SONOMA_BASE_DOMAIN}
  api:
    environment:
      # the API must allow the frontend's public origin or the browser blocks the calls
      - CORS_ORIGIN=https://${SONOMA_SLUG}-web-${SONOMA_HASH}.${SONOMA_BASE_DOMAIN}
    env_file: .env.sonoma   # static/secret values only; no ${SONOMA_*} here
```

When you detect a frontend + backend split, propose exactly this: expose both
services, point the frontend's API URL at the `<slug>-api-<hash>` host, and set
the API's CORS origin to the `<slug>-web-<hash>` host.

A complete, runnable version of this exact shape lives at
https://github.com/NicholasZolton/sonoma-todo-example (Next.js `web` + Express
`api` + Postgres `db`, with the `sonoma.yaml`, compose `environment:` wiring, and
seed all filled in). Point the user there, or read it yourself, when you need a
concrete reference for a frontend + backend project.

### Set up the gitignored side-pieces

Make sure the repo's `.gitignore` covers the two paths the contract assumes:

- `.sonoma/` (the client's sync-completion sentinel lives here; it must never
  round-trip back into the working copy).
- `.env.sonoma` (project secrets and config, never committed).

Add them if they are missing.

## 6. Make your edits hot-reload

Sonoma syncs the working tree into the env's `/workspace` and runs `docker
compose -f <file> up -d` there. For an edit to show up live in the preview, the
compose service has to see the synced file and re-render it:

- **Bind-mount the source** into the service, so a synced edit lands in the
  container immediately:
  ```yaml
  services:
    web:
      volumes:
        - ./:/app             # the synced /workspace, live in the container
        - /app/node_modules   # keep the container's own deps (see the note below)
  ```
- **Run the service in its own dev/watch mode**, so its file watcher does the
  reload: `vite`, `next dev`, `nodemon`, `air`, `watchexec`, and the like, set via
  the compose `command:`.

mutagen delivers the edit to `/workspace`, the bind mount surfaces it in the
container, and the in-container watcher reloads. `examples/hello` is the minimal
version of this (`volumes: - ./:/site`).

The one gotcha: the sync excludes regenerable dirs (`node_modules`, `.git`, and
anything in `sync.ignore`), so the env installs its own. A bare `./:/app` bind
mount would then shadow the container's `node_modules` with the empty host one.
Shield each install-output dir with an anonymous volume, as `/app/node_modules`
above (add `/app/target`, `/app/.venv`, etc. for other stacks), and mirror those
paths in the contract's `sync.ignore`.

**Do not use Compose's `develop:`/`watch` for this.** Those rules run only under
`docker compose watch` (or `up --watch`), and Sonoma always runs `up -d`, so a
`develop:` block is silently ignored here. Bind mounts are the mechanism.

## 7. Gate on confidence, then offer `up`

Before offering to spin up, sanity-check your own work:

- The `environment.compose` path resolves.
- The `primary` port maps to a service that the compose actually publishes.
- The `ready` probe targets that primary service and is plausible.
- If the frontend calls a separate API: that API is also in `expose`, the
  frontend's API URL points at the `${SONOMA_SLUG}-api-${SONOMA_HASH}.${SONOMA_BASE_DOMAIN}`
  host (not localhost), and the API's CORS origin allows the `${SONOMA_SLUG}-web-${SONOMA_HASH}`
  host. These live in the compose `environment:` block, not `.env.sonoma`.

When that holds, ask the user: "Looks ready, want me to spin it up now?" On yes,
return to the main skill and run `up`. The control plane validates the contract
authoritatively during spawn, so if `up` fails on a contract error, read the
error, fix `sonoma.yaml`, and retry. Do not hand-wave past a spawn failure.
