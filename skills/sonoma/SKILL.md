---
name: sonoma
description: Use when the user wants an isolated, seeded full-stack preview environment for a coding task. Spawns a remote env, syncs the working tree, and returns a shareable preview URL.
---

# Sonoma

Spin up an isolated full-stack environment for the current task and return a
preview URL. The environment runs on the Sonoma control plane; your edits sync
into it live, so you can QA against the preview while you work.

How it fits together: the Sonoma CLI is the deterministic local driver, the
control plane hosts the env and validates the project contract, and each repo
boots from a small contract. Sonoma reads that contract from a `sonoma.yaml`
when the repo has one, and otherwise infers it from a `docker-compose.yml`: the
ports the services publish become the preview URLs. The first time a repo is
used it needs a one-time local setup (healthy deps and login), plus a
`sonoma.yaml` only when the repo has no compose file; after that, every task
just spawns an env.

## Steps

The Sonoma CLI is the `sonoma` command, installed globally from the npm package `@sonoma-sh/cli` (it runs on the Bun runtime). If `sonoma` is not on PATH, ensure bun is present (`curl -fsSL https://bun.sh/install | bash`) and install the CLI with `bun add -g @sonoma-sh/cli`, then continue.

1. Confirm the project and branch. The project is this git repository; the
   branch is the current git branch. The user may override either.
2. **First run only:** get the local setup healthy, and put a contract in place
   only if the repo needs one. Read [onboarding.md](./onboarding.md) and follow
   it: install the CLI, run `sonoma doctor` until green, and log in. A repo with
   a `docker-compose.yml` needs no `sonoma.yaml`. `sonoma up` reads the ports its
   services publish and builds the contract itself. Author a `sonoma.yaml` only
   when the repo has no compose file, or to override the inferred seed, readiness
   probe, or primary service. Once setup is done, skip straight to the next step.
3. Spawn (or reuse) the environment and start the sync:

   ```bash
   sonoma up
   ```

   This prints the env id and, once ready, the preview URL. If it reports that
   you are not logged in, tell the user to run `sonoma login` (it opens the browser to sign in) and stop.
   If `up` hangs on a phase (for example it sits at `AWAITING_SYNC`) or the env
   reports `FAILED`, do not just retry blindly: read
   [debugging.md](./debugging.md) and work through it. You can run commands and
   stream logs inside the env to find the cause.
4. Do the coding work for the user's request in this repository as usual. The
   running sync delivers every edit into the environment, so the preview
   reflects your changes without any extra step.
5. QA against the preview URL (load it, exercise the changed flow). Report the
   preview URL and a short summary of what you changed and verified.
6. The environment is torn down automatically when its branch's PR merges. If
   the user wants to end it sooner, run `sonoma down`.

Keep this skill thin. The orchestration lives in the `sonoma` CLI and the
control plane, not here.
