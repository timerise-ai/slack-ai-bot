# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.3] - 2026-09-28

Fix release, from scoring the prompt-1 agent eval runs against 0.1.2.

### Fixed

- The suite in `references/testing.md` names `lib/slack-bot/slack-bot.test.ts`
  as its destination but imported `../lib/slack-bot/...`, so it did not compile
  there and every agent had to move or edit it. Its imports are now relative to
  its own path. Apps built from earlier versions need no change: an edited copy
  that compiles runs the same 31 tests.

### Changed

- `SKILL.md` quick start: copy every code block verbatim, write the whole file
  map even for an inbound-only task, and know which parts are yours to write
  (`host.ts`, the renames, the `BOT_STRINGS` text, a non-Supabase store). A new
  last step names what the handover must tell the operator.
- `SKILL.md` critical facts and hard rules: never fall back to memory state or
  memory stores on a missing variable; identity is the Slack email or the
  user-confirmed link; a bot token is a row, never an environment variable.
  The README's third non-negotiable says the same.
- `references/setup.md`: the env table is exactly what goes into
  `.env.example`, empty and tracked; "read every credential from the
  environment" does not cover bot tokens; the chat state adapter and the stores
  are chosen unconditionally and fail at first use when their URL is missing.
- `references/testing.md`: the `bun:test` import is the only line that changes;
  the suite is never moved, split or converted, and a missing runner is
  installed from the registry.
- `references/adaptation.md`: an administrator-entered mapping is not one of
  the two identity variants.

## [0.1.2] - 2026-09-28

Documentation release. The skill content is unchanged from 0.1.1.

### Added

- `evals/prompts.md`: three prompts an operator types after installing, the
  first of which the agent evals run before every release.
- `.github/workflows/agent-eval.yml`: the caller of the index's reusable eval
  workflow, run on every published release and on a maintainer's dispatch.

### Changed

- The README file table lists every file in the repository, `evals/` and the
  eval workflow included.
- `CLAUDE.md` describes `evals/` and the eval workflow, and records that evals
  are not skill content.

## [0.1.1] - 2026-09-21

Wording release. The skill content is unchanged from 0.1.0.

### Added

- `SKILL.md` closes with a line linking the
  [Timerise Skills](https://github.com/timerise-ai/skills) index, so an agent that
  has the skill loaded can find the sibling skills for neighbouring modules without
  leaving the entry point.

### Changed

- `CLAUDE.md` records the closing line in the `SKILL.md` layout, and the line budget
  it states holds that line aside.

## [0.1.0] - 2026-09-21

First release. A two-way Slack AI bot for a **Next.js App Router** app: people
mention or DM the bot and a model answers with tools scoped to their app
account, the app posts into channels on its own, and risky actions wait behind
Approve and Cancel buttons whose clicks come back to the app and run the side
effect exactly once.

### Added
- `SKILL.md`: the entry point. Architecture diagram, eight critical facts, five
  hard rules, the quick-start order and the reference directory table.
- `references/adaptation.md`: the seam contract with the host app, the rename
  table, the `BotHost` interface with a demo implementation, the identity
  lookup and its trust decision, the host probe.
- `references/data-model.md`: `slack_installations` and `slack_approvals`, the
  store interfaces, the Postgres schema, Supabase and in-memory
  implementations, and the conditional-update `claim` in four data layers.
- `references/setup.md`: Slack app configuration, environment, the requested
  scope list, the OAuth install routes, chat state, the cached bot factory and
  the events route.
- `references/conversation.md`: identity resolution with its cache, origin
  parsing, the handlers, the user-scoped tools and the system prompt.
- `references/outbound.md`: the Slack Web API client with its limits and 429
  retry, report posting, the Markdown to mrkdwn converter and the strings
  block.
- `references/approvals.md`: the approval service and its state machine,
  signature verification, and the interactivity route.
- `references/testing.md`: 31 tests over the pure logic, what each group
  proves, and the manual end-to-end script.
- `references/operations.md`: the symptom table, stuck approvals, uninstalls,
  what to make visible, logging, and the go-live checklist.
- `references/provenance.md`: the engineering ledger. What the audit of the
  earlier implementation changed and how the templates verify it, what was kept
  deliberately, and what was designed here and has never run in production.
- `README.md`, `CLAUDE.md`, `LICENSE`.
