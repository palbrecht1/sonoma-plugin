# Debugging a stuck or failed env

Use this when `sonoma up` hangs on a phase, the preview URL never comes up, or an
env reports `FAILED`. The whole point: you can run commands inside the env and
stream its logs, so you almost never have to guess.

## The toolbox

Read the complete [CLI command reference](./cli-reference.md), especially
*Environment access and debugging*. It contains the exact current help, targeting
behavior, examples, and guidance for `status`, `exec`, `ssh`, `sync`, and
`down`. Prefer `sonoma exec [id] -- <cmd>` for reproducible agent work and use
`sonoma ssh [id]` when an interactive shell is genuinely useful.

`exec` and `ssh` land you in `/workspace`, which is the env's WORKDIR and the
root the sync writes into. The env is a small Alpine image with `docker`,
`docker compose`, `git`, `curl`, and `bash` available.

## Know which phase you are in

An env boots **empty**. Your working tree arrives by a mutagen sync from your
machine, not by the server cloning the repo. It then moves through:

```
PROVISIONING  ->  AWAITING_SYNC  ->  SEEDING  ->  READY
 (VM boots)      (your tree lands)  (compose up,   (preview URL live)
                                     seed, probe)
```

or lands in `FAILED` with a reason. `sonoma status <id>` prints that reason on a
`↳` line, so start there. The env's sshd answers from `AWAITING_SYNC` onward, so
`exec` and `ssh` work while it is still coming up, not just once it is `READY`.

When available, `status` also prints `configuration source: request`,
`imported`, or `synthesized`. This identifies the contract source only; status
does not retain a repository key. Fresh `up` output includes the repository key
used for that spawn.

## Running commands and streaming logs

The stack runs inside the env as `docker compose -f <compose-file> up -d` from
`/workspace`, where `<compose-file>` is your contract's `environment.compose`
(or `docker-compose.yml` when the contract was inferred). So:

```bash
# what containers exist and their state
sonoma exec -- docker compose -f docker-compose.yml ps
sonoma exec -- docker ps -a

# stream service logs (drop -f for a one-shot dump, add a service name to narrow)
sonoma exec -- docker compose -f docker-compose.yml logs -f
sonoma exec -- docker compose -f docker-compose.yml logs --tail=100 api

# poke around interactively
sonoma ssh
```

## Stuck at AWAITING_SYNC (the common one)

`AWAITING_SYNC` means the env is up and waiting for your code to finish landing.
The gate is a single sentinel file: after `mutagen flush` delivers the whole
tree, the client creates `/workspace/.sonoma/sync-complete`, and the control
plane polls for it before it runs `compose up`. Stuck here means the tree never
fully landed, the sentinel never got dropped, or the mutagen session is wedged.
If it never resolves, the phase times out and the env goes `FAILED` with
"project code did not sync ... is the mutagen session connected?".

Work through it:

1. **Confirm the state.** `sonoma status <id>`. A flush in progress can look
   stuck for a moment, so give it a beat. If it already shows `FAILED` with the
   sync message, the session never connected.
2. **Did the tree arrive?** `sonoma exec <id> -- ls -A /workspace` should list
   your repo. An empty or partial `/workspace` means the sync did not deliver.
3. **Is the sentinel there?** `sonoma exec <id> -- ls -A /workspace/.sonoma`.
   Files present but no `sync-complete` means the flush landed but the sentinel
   was never dropped.
4. **Fastest fix: re-run `sonoma up`.** It recreates the mutagen session,
   flushes the full tree, and re-drops the sentinel. This clears most cases.
5. **Reset a wedged session:** `sonoma sync stop <id>` then `sonoma up`.
6. **Push it by hand** when the files are present but the sentinel is not:
   ```bash
   sonoma sync flush <id>
   sonoma exec <id> -- sh -c 'mkdir -p /workspace/.sonoma && touch /workspace/.sonoma/sync-complete'
   ```
   That mirrors exactly what `sonoma up` does to signal completion.

Note: Sonoma runs mutagen under a per-env data directory with a wrapped ssh, so
a bare `mutagen sync list` in a plain shell will not see these sessions. Use the
`sonoma sync` verbs.

## Stuck or failed in SEEDING (compose up, seed, readiness)

`SEEDING` covers three sub-steps: bringing the stack up, running your `seed`,
then retrying the `ready` probe until it passes. Failures here are almost always
the project's own stack, and the logs tell you which.

1. `sonoma status <id>` and read the `↳` reason (for example, "readiness probe
   never passed" or a compose error).
2. Check container state and logs with the commands under *Running commands and
   streaming logs* above. A crash-looping service shows up here immediately.
3. **Reproduce the failing step by hand:**
   - readiness: run the exact `ready` command from `sonoma.yaml`, for example
     `sonoma exec <id> -- curl -fsS http://localhost:3000/healthz`.
   - seed: run the `seed` command yourself and watch it fail.
4. `sonoma ssh <id>` for anything deeper (env vars, file contents, a manual
   `docker compose up` without `-d` to watch it boot in the foreground).

Remember the shareable-URL rules from onboarding: if the preview loads but its
API calls fail, the frontend is probably calling `localhost` instead of the
`<slug>-api-<hash>` host, or the API's CORS origin does not allow the
`<slug>-web-<hash>` host. That is a contract/compose fix, not an env fault.

## Failed at PROVISIONING

The VM never booted. This is the control plane's side, so there is no env to
`exec` or `ssh` into. `sonoma status <id>` shows the reason. Re-run `sonoma up`
to retry once; if it keeps failing at provisioning, it is a server issue, so
surface it to the user rather than looping.

## Cleaning up and retrying

- `sonoma down [id]` tears an env down; re-run `sonoma up` for a fresh one.
- A `FAILED` env is garbage-collected to `DESTROYED` after a retention window,
  and `DESTROYED` envs are hidden from `sonoma status` unless you pass `--all`.
  So read the failure reason while it is still listed, or spawn again.
