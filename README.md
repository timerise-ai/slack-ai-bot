# slack-ai-bot

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to build a two-way Slack AI bot inside a
**Next.js App Router** app, with the Chat SDK and the AI SDK. People mention the bot, DM it or reply in a
thread it follows, and a model answers with tools scoped to their account in the app. The app posts reports
and alerts into channels on its own schedule. Risky actions wait behind Approve and Cancel buttons, and the
click comes back as a webhook that runs the side effect exactly once.

**Slack is an untrusted, stateless, retrying client, and the model is an untrusted caller.** Every id the
model passes into a tool came from whoever was talking in the thread. Every webhook may arrive twice, on a
cold instance, with three seconds to answer before Slack tells the user it timed out and they click again.
That is where the work is: identity, scoping, acknowledgement and idempotency. The chat loop itself is twenty
lines.

This skill was written by the engineer who has shipped this module. The earlier implementation it was audited
against was the Slack assistant of a product on Next.js 16, Vercel and Supabase. The templates hold the
properties a two-way bot has to hold: every answer and every tool call scoped to the person who asked, one
side effect per approved click however often the button is clicked, both webhooks acknowledged inside Slack's
three seconds, a bot token chosen by the workspace the event came from, and a failure the person in the thread
can see. The suite in [`references/testing.md`](references/testing.md) states each one;
[`references/provenance.md`](references/provenance.md) has the record.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/slack-ai-bot
```

Name the agents instead with `-a`, for example
`npx skills add timerise-ai/slack-ai-bot -a claude-code -a codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md` plus markdown references with no file that calls a model, so cloning it into an agent's skills
directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/slack-ai-bot.git ~/.claude/skills/slack-ai-bot
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/slack-ai-bot ~/.agents/skills/slack-ai-bot
```

Update the skill with `git pull` in its directory. The current release is **0.1.2**. See
[CHANGELOG.md](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: adding a Slack assistant that answers
from a product's own data, pushing app-initiated messages into Slack, putting a human approval step in front
of an action a model would otherwise fire, or auditing a bot that misses thread follow-ups, answers twice or
cannot identify the person talking to it. Invoke it explicitly with `/slack-ai-bot` in Claude Code,
`$slack-ai-bot` in Codex CLI, or from `/skills` in Gemini CLI.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: architecture diagram, critical facts, hard rules, quick start, and the reference directory |
| `README.md` | This file |
| `CHANGELOG.md` | One section per release, newest first |
| `CLAUDE.md` | The editing conventions, for an agent editing this repository |
| `LICENSE` | MIT |
| `references/adaptation.md` | The seam contract: the `BotHost` interface, the rename table, the identity lookup and its trust decision, the host probe |
| `references/data-model.md` | `slack_installations` and `slack_approvals`, the store interfaces, the Postgres schema, and the conditional-update `claim` in four data layers |
| `references/setup.md` | Slack app configuration, environment, the requested scope list, OAuth install routes, chat state, the cached bot factory, the events route |
| `references/conversation.md` | Identity resolution and its cache, origin parsing, the handlers, the user-scoped tools, the system prompt |
| `references/outbound.md` | The Web API client and Slack's limits, report posting, the Markdown to mrkdwn converter, the strings block |
| `references/approvals.md` | The approval service and its state machine, signature verification, the interactivity route |
| `references/testing.md` | The suite, 31 tests, what each group proves, and the manual end-to-end script |
| `references/operations.md` | Symptom table, stuck approvals, uninstalls, what to make visible, logging, the go-live checklist |
| `references/provenance.md` | The engineering ledger: what the audit changed and how the templates verify it, what was kept on purpose, and what is new in the skill |
| `evals/` | The prompts an operator types after installing (`prompts.md`) and one file per agent eval: the skill installed into an empty Next.js app, one prompt, no help, then type-checked, built and tested |
| `.github/workflows/agent-eval.yml` | The caller of the index's reusable eval workflow, run on every published release and on a maintainer's dispatch |

The seam is the contract table at the top of `references/adaptation.md` and the `BotHost` interface under it:
one file the host app fills in. It bounds the domain nouns, the app's own user id and access check, the
identity lookup by email, the two stores, the chat state adapter, the model, the executors that run an
approved action, and the strings the bot posts. Everything above that interface is the skill's; everything
below it is the host app's, including auth, tenancy, the ORM and i18n. Slack renders the UI, so there is no
styling seam at all.

## The five non-negotiables

These travel with the module and are never optional. Each is stated as a hard rule in `SKILL.md` and covered
by the suite in `references/testing.md`:

1. **Never put a payload in a button `value`.** Everyone in the channel can read what a button carries, and a
   client can send any value back. The button carries an opaque approval id and the rest is loaded
   server-side. A test asserts the posted blocks contain the id and not the payload.
2. **Never authorize a click by channel membership.** Anyone who can see a message can click it, so the
   clicker is resolved to an app user and checked against the resource, every time. A test clicks as an
   outsider and asserts the request is refused and still pending.
3. **Never pick a bot token with `limit(1)`.** A token belongs to a workspace, not to whoever pressed Install,
   so `slack_installations` is keyed by `team_id` and the team comes off the event. One row per workspace is
   the shape the schema enforces.
4. **Never parse the body before verifying the signature.** The HMAC is over the raw bytes, so a re-serialized
   body never matches. `request.text()` comes first, and the comparison is over byte lengths, so a forged
   multi-byte signature returns false instead of throwing. Both are in the suite.
5. **Never let a failure be silent.** After the eyes reaction, silence reads as "still working". A thrown
   handler posts into the thread, and a refused or failed click answers the clicker through `response_url`.

Everything else is the host app's: its users table, its auth, its ORM, its model, its domain nouns and the
language the bot speaks.

## Requirements

- **Next.js App Router** on a Node runtime, with a way to run work after the response: `after()` from
  `next/server`, `waitUntil` from `@vercel/functions`, or a queue.
- `chat` and `@chat-adapter/slack`, a chat state adapter, `ai` and `zod`. In production the state adapter must
  be shared, Redis or Postgres: with in-memory state every cold start forgets which threads the bot follows.
- A Slack app you control, distributed by OAuth, with the scope list in `references/setup.md` requested by the
  authorize URL rather than only listed on the config page.
- A users table that can be queried by lower-cased email in one indexed lookup. The Slack profile email is the
  identity bridge, and `references/adaptation.md` spells out the trust decision that implies and what to do
  instead when it is not acceptable.

## Verification

The pure logic carries 31 tests, written for `bun:test` and one import line away from vitest. The templates
type-check under `strict` and `--noUncheckedIndexedAccess` against `chat` 4.40, `ai` 7, `zod` 4, `next` 16 and
`@supabase/supabase-js` 2. What is not covered is stated rather than implied, in `references/testing.md`: the
SQL is not executed, the state adapter and Drizzle or Prisma snippets are not compiled, the handlers, the bot
factory and the routes are type-checked only, and nothing here talks to a real workspace. The manual
end-to-end script at the end of that file is the pass against a live Slack app.

## Not this

| Not this | Use instead |
|---|---|
| Learning the Chat SDK: cards, modals, other platforms | The `chat-sdk` skill. This skill uses the SDK, it does not document it |
| Model choice, streaming internals, tool-calling mechanics | The `ai-sdk` skill |
| Reading channel history as a data source | A history module of its own: history scopes, pagination and user-name resolution are not here |
| One-way notifications with no reply path | A Slack incoming webhook URL. No bot user, no tokens table, no webhooks to verify |
| Slash commands or modals as the main interface | The `chat-sdk` skill; this module is mentions, DMs, threads and buttons |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build step, but the code blocks are checked:
every TypeScript block names its destination on the first line, and the module and test blocks are written to
compile as one project under `strict` and `noUncheckedIndexedAccess` and to run under `bun test`, 31 tests.
Claims in this skill are meant to be verifiable: if you change a factual claim, say how you verified it,
whether against the Slack Web API documentation, the `chat` adapter's own code, the AI SDK, or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of
the templates is there for a reason, and `references/provenance.md` is the ledger that must stay truthful:
read it before simplifying anything, and add an entry for anything you change. Commits follow Conventional
Commits and releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the
index; `CLAUDE.md` carries the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
