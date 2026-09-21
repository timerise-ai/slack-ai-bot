# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json` here and nothing
in this repository executes. It teaches an agent to build a two-way Slack AI bot inside a **Next.js App
Router** app: inbound answers from a model with tools scoped to the asking user, app-initiated posting into
channels, and an approval round trip where a button click comes back as a webhook and runs a side effect
exactly once.

Keep the two straight: the commands and code in `references/` describe the app the agent will generate, not
this repository. The SQL in `data-model.md` and `adaptation.md`, the `curl` check and the stuck-approval query
in `operations.md`, the host probe in `adaptation.md` and the `bun test` invocation in `testing.md` all run in
that generated app. The one thing checked here is that the templates compile and their tests pass, and that
check runs in a scratch project; the recipe is under *Editing conventions* below.

The skill was written by the engineer who has shipped this module; the earlier implementation it was audited
against was the Slack assistant of a product on Next.js 16, Vercel and Supabase. `references/provenance.md` is
the ledger of that audit: fourteen entries on what changed and how the templates verify it, what was kept
deliberately, and what was designed here and has never run in production. That file is the rationale layer:
read it before "simplifying" anything.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines. The
  frontmatter `description` is the trigger surface; the body carries the architecture diagram, eight
  **critical facts**, five **hard rules**, the quick-start order, and the **reference directory table**
  mapping trigger keywords to files.
- `README.md`: the human-facing front door, in the section order of the skill standard: install, activation,
  the file table, the five non-negotiables, requirements, verification, the *Not this* table, contributing.
- `references/*.md`: one topic per file, loaded on demand. `adaptation.md` (the seam contract and `BotHost`)
  and `data-model.md` (the two tables and their stores) are the design entry points; `setup.md` carries the
  Slack app, OAuth and boot; `conversation.md` the inbound half; `outbound.md` the app-initiated half;
  `approvals.md` the round trip; `testing.md` the suite and the manual script; `operations.md` the operator
  surface; `provenance.md` the audit.

## Editing conventions

- **Code blocks name their destination on the first line** as a comment, for example
  `// lib/slack-bot/host.ts`. That line is what makes a block extractable, so keep it and keep imports
  complete. A block that continues a file already introduced omits it.
- **The code blocks are compiled and run.** The TypeScript blocks across `adaptation.md`, `data-model.md`,
  `setup.md`, `conversation.md`, `outbound.md`, `approvals.md` and `testing.md` form one project: write each
  to the path named on its first line in a scratch directory, install what they import (`next`, `chat`,
  `@chat-adapter/slack`, a state adapter, `ai`, `zod`, `@supabase/supabase-js`, `typescript`), then

  ```bash
  npx tsc --noEmit          # strict, noUncheckedIndexedAccess, skipLibCheck, paths {"@/*": ["./*"]}
  bun test lib/slack-bot    # 31 tests; under vitest, change the one bun:test import
  ```

  `skipLibCheck` is not optional, or Next's own type declarations fail the run and say nothing about these
  templates. Re-run after editing any block. The snippets marked as unverified are marked for a reason and
  stay outside that project: the SQL in `data-model.md` and `adaptation.md`, the `findUserIdByEmail` body
  fragment in `adaptation.md`, the Redis and Postgres state adapter variants in `setup.md`, and the Drizzle,
  Prisma, raw SQL and Firestore `claim` rows in `data-model.md`.
- **Identifiers are shared across files.** `BotHost`, `botHost`, `ResourceSummary`, `ResourceDetail`,
  `ActionContext`, `ActionOutcome`, `InstallationStore`, `ApprovalStore`, `ApprovalRecord`, `NewApproval`,
  `ApprovalExecutor`, `ApprovalService`, `SlackOrigin`, `createApprovalService`, `createSlackUserResolver`,
  `extractSlackOrigin`, `verifySlackSignature`, `markdownToMrkdwn`, `slackApi`, `splitIntoChunks`,
  `buildTextBlocks`, `postBlocks`, `updateBlocks`, `postReport`, `respondEphemeral`, `safeReturnPath`,
  `withParam`, `buildAuthorizeUrl`, `exchangeCode`, `getBot`, `registerHandlers`, `createBotTools`,
  `SLACK_BOT_SCOPES`, `OAUTH_STATE_COOKIE`, `APPROVE_ACTION_ID`, `CANCEL_ACTION_ID`, `BOT_STRINGS`, and the
  env names `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_SIGNING_SECRET`, `NEXT_PUBLIC_APP_URL` appear in
  several references. Rename in all of them or none.
- **Keep the three tables in sync** with `references/`: the reference directory in `SKILL.md`, the quick-start
  list in `SKILL.md`, and the file table in `README.md`. Links are relative: `[x.md](references/x.md)` from
  `SKILL.md`, `[x.md](x.md)` between references.
- **Do not remove the odd-looking parts.** The cached init promise rather than a boolean flag; the approval
  row written before the message is posted; `claim` as one conditional `UPDATE`; the access check placed
  before the claim; the byte-length guard before `timingSafeEqual`; identity resolved before any reaction is
  added; the silent identity failure in a followed thread; `isBot === true` rather than a truthiness check;
  the placeholder parking in the converter; `installationProvider` instead of seeding chat state; the `null`
  return for an origin with an empty id. Each is a ledger entry or a documented judgement call. Check
  `provenance.md` before touching one.
- **The numbers that remain are load-bearing.** 31 tests, fourteen ledger entries, Slack's own limits (3000
  characters per section, 50 blocks, 150 for a header, about 4000 of fallback text, a 5-minute signature
  skew, a 3-second ack), and the design parameters (a 10-minute identity TTL, a six-step tool budget, an
  800 ms streaming interval, a 1500-character preview, an 8000-character tool result cap, `maxDuration` 300).
  They were verified against this repository, against Slack's documentation, or they are parameters the next
  implementation needs. Do not restate them loosely and do not add new ones. Figures describing the earlier
  implementation's deployment do not appear anywhere.
- **Mark additions as additions.** Anything designed in the skill and never run in the earlier implementation
  belongs in the "Added" section of `provenance.md`, stated as such, or marked in place in the reference the
  way the 429 retry and the `response_url` reply are. The skill's credibility is that it distinguishes the
  two.
- **Never present the non-negotiables as optional.** The opaque button value, the clicker resolved and
  authorized, the token keyed by workspace, the signature verified over the raw body, and the failure the
  person can see are stated as hard rules in `SKILL.md` and as non-negotiables in `README.md`; keep them that
  way everywhere.
- **Which names the host renames**, and which are the authoring contract: `Resource`, `resourceId`,
  `search_resources`, `get_resource`, `run_action`, `Approval` and `kind` are canonical vocabulary the host
  renames to its own nouns, and `adaptation.md` carries that table; the bot's name and every string in
  `BOT_STRINGS` are the host's too. Slack's own terms (`team_id`, `channel`, `thread_ts`, `ts`, `action_id`,
  `block_actions`, `response_url`), the env var names and the `BotHost` method signatures are the contract and
  are not renamed.
- **The prose is plain ASCII.** No em-dashes, arrows, middle dots or smart quotes, including in the diagrams,
  which are drawn with `-`, `|`, `+`, `v` and `^`. Two exceptions are code, not prose, and must stay exactly
  as they are: the bullet and divider characters the mrkdwn converter emits in `outbound.md`, and the
  multi-byte character in the forged-signature test in `testing.md`, which is the whole point of that test.
