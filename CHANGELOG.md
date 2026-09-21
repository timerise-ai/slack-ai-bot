# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
