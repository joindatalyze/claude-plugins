# Datalyze plugins for Claude Code

This marketplace holds one plugin, `datalyze`, the [Datalyze Tracking Agent](https://www.joindatalyze.com/tracking-agent). It audits a product's analytics tracking, says what tracking each change needs, and builds the changes a person approves. It needs a Datalyze subscription: each run fetches Datalyze's tracking knowledge with your workspace key.

## Install

```
/plugin marketplace add joindatalyze/claude-plugins
/plugin install datalyze@datalyze
```

Then set `DATALYZE_KEY` to the key on your Datalyze dashboard before you start Claude Code. Teams set up through Datalyze already have the marketplace in the repo's `.claude/settings.json`, so Claude Code offers the plugin when you open the repo.

The plugin's tools need Node 22 or later.

## Use

Ask Claude Code in your own words, or use the commands:

| Ask | Command |
|---|---|
| "Audit our tracking", "audit the checkout events" | `/datalyze:audit` |
| "What does this branch need for tracking?" | `/datalyze:review` |
| "Implement the approved proposal" | `/datalyze:implement` |

What the plugin writes: the audit to `analytics/.datalyze/audit.json` and `audit.md`, a branch's proposal to `.datalyze/proposals/`, and approved changes to your tracking plan and code. Your code, data and analytics keys stay on your machine or runner; only the key check goes to Datalyze.

## This repository

The plugin here is built from Datalyze's own source and synced in; `bin/` is a bundle, committed because a plugin install has no build step. Please send changes and issues to ansh@joindatalyze.com rather than as pull requests.
