# Sonoma onboarding (first run in a repo)

Follow this when the user wants a Sonoma environment but the repo has **no
`sonoma.yaml` at its root**. The goal is to get the local setup healthy, author
the project contract with the user, and only then offer to spin up the env.

The CLI is the `sonoma` command (it runs on the Bun runtime). Do the steps in
order. Do not skip ahead to `up`.

## 1. Make sure the CLI is installed

Check whether `sonoma` is on PATH:

```bash
command -v sonoma
```

If it is missing, ensure bun is present, then install the CLI globally:

```bash
curl -fsSL https://bun.sh/install | bash   # only if bun is missing
bun add -g @sonoma-sh/cli
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

## 4. Author the project contract (`sonoma.yaml`)

This is the part the CLI does not do for you. Detect the project's shape, propose
a contract, confirm it with the user field by field, then write the file.

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
- `expose`: one entry per service the user reaches. Mark the user-facing one
  `primary: true`. That primary is both the share link and the readiness target.
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
expose:
  - service: web                          # must match a service in the compose
    port: 3000
    primary: true                         # the share link + readiness target
  - service: api
    port: 4000
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

### Shareable URLs: the slug, cross-service APIs, and CORS

This is the part that makes the preview, and any API the frontend calls,
reachable from a browser. Get it right or the preview loads but its API calls
fail.

How addressing works:

- Each env gets a short id. Sonoma sets two environment variables on the VM
  where your compose runs, and these are the only handoff:
  - `SONOMA_SLUG`: the env's short id.
  - `SONOMA_BASE_DOMAIN`: `sonoma.sh`.
- Every service you list in `expose` is reachable at a single-level subdomain
  `https://<service>--${SONOMA_SLUG}.${SONOMA_BASE_DOMAIN}` (one wildcard cert
  covers all of them). The `primary` entry is the shareable link.

What this means for `expose`: add an entry for every service that is hit from
outside the env, not just the frontend. A browser-side frontend that calls a
separate backend means **both** `web` and `api` need to be exposed, because the
browser reaches the API over its public `api--<slug>` URL, not `localhost`.

The interpolation rule that trips people up:

- `.env.sonoma` is loaded via compose `env_file:`, and those values are
  **literal**, so `${SONOMA_SLUG}` does NOT expand there. Put only static or
  secret values in `.env.sonoma`.
- Anything that must reference the slug (the API URL the frontend points at, and
  the CORS allow-origin the API honors) goes in the compose `environment:`
  block, which compose **does** interpolate, because Sonoma set those vars on the
  VM.

So in the project's `docker-compose.yml`:

```yaml
services:
  web:
    environment:
      # the frontend (in the browser) calls the API at its public URL, not localhost
      - NEXT_PUBLIC_API_URL=https://api--${SONOMA_SLUG}.${SONOMA_BASE_DOMAIN}
  api:
    environment:
      # the API must allow the frontend's public origin or the browser blocks the calls
      - CORS_ORIGIN=https://web--${SONOMA_SLUG}.${SONOMA_BASE_DOMAIN}
    env_file: .env.sonoma   # static/secret values only; no ${SONOMA_*} here
```

When you detect a frontend + backend split, propose exactly this: expose both
services, point the frontend's API URL at the `api--<slug>` host, and set the
API's CORS origin to the `web--<slug>` host.

### Set up the gitignored side-pieces

Make sure the repo's `.gitignore` covers the two paths the contract assumes:

- `.sonoma/` (the client's sync-completion sentinel lives here; it must never
  round-trip back into the working copy).
- `.env.sonoma` (project secrets and config, never committed).

Add them if they are missing.

## 5. Gate on confidence, then offer `up`

Before offering to spin up, sanity-check your own work:

- The `environment.compose` path resolves.
- The `primary` port maps to a service that the compose actually publishes.
- The `ready` probe targets that primary service and is plausible.
- If the frontend calls a separate API: that API is also in `expose`, the
  frontend's API URL points at the `api--${SONOMA_SLUG}.${SONOMA_BASE_DOMAIN}`
  host (not localhost), and the API's CORS origin allows the `web--${SONOMA_SLUG}`
  host. These live in the compose `environment:` block, not `.env.sonoma`.

When that holds, ask the user: "Looks ready, want me to spin it up now?" On yes,
return to the main skill and run `up`. The control plane validates the contract
authoritatively during spawn, so if `up` fails on a contract error, read the
error, fix `sonoma.yaml`, and retry. Do not hand-wave past a spawn failure.
