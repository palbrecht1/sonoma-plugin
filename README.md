# Sonoma — Claude Code plugin

The `/sonoma` skill: spin up an isolated, seeded, full-stack preview environment
for a coding task and get a shareable preview URL, all from Claude Code.

> This repository is a published mirror. It is generated from the `plugin/`
> directory of the (private) Sonoma monorepo on each `plugin-v*` release. Do not
> edit it directly, changes here are overwritten by the next sync.

## Install

```
/plugin marketplace add palbrecht1/sonoma-plugin
/plugin install sonoma@sonoma
```

The skill drives the `sonoma` CLI, installed globally from npm:

```
bun add -g @sonoma.sh/cli
```

On first use in a repo, the skill walks you through one-time setup (login,
`doctor`, and authoring a `sonoma.yaml` contract). After that, every task just
runs `sonoma up` to get a preview URL.
