# Provenance

The engineering ledger behind the templates, for the person editing this skill.
It keeps three things apart: what the audit of the earlier implementation
changed and how the templates verify it, what was kept deliberately and why it
is safe, and what was designed here and has never run in production.

The earlier implementation was the Slack assistant of a Next.js 16 application
on Vercel and Supabase: it answered questions about the asking user's own
records, ran work on request, and posted drafts for a human to approve in a
thread. The architecture, the handler flow, the tool-scoping pattern, the
approval round trip and the block-splitting come from it. Its domain code stayed
in the host app, where it belongs.

Every defect below was confirmed by reading that code. Two were also confirmed
by execution: the Markdown converter's outputs and the signature comparison's
thrown error. Production impact was **not** observed; the consequences stated
are what the code does, not incident reports.

## Fixed in the templates

### 1. Two read tools returned any tenant's data by id
Both queried with the service-role client and never checked that the asking user
could see the record. The tool that ran the action did check.
**Shipped:** `BotHost` has no id-only method; every lookup takes `userId`, see
[conversation.md](conversation.md), [adaptation.md](adaptation.md).

### 2. The install flow did not request the scopes identity depends on
The authorize URL omitted `users:read.email`, `app_mentions:read`, `im:read`
and `im:write`, though the setup docs listed them. Tokens from the app's own
Connect button cannot read emails, so no user can be matched.
**Shipped:** one exported scope list, with a test, see [setup.md](setup.md).

### 3. In-memory chat state in a serverless deployment
Subscriptions, dedupe and locks were per-instance. Thread follow-ups are dropped
whenever they land on an instance that did not see the first mention.
**Shipped:** state is a seam with production adapters named, see [setup.md](setup.md).

### 4. Approve could send twice
Status was checked in application code, the email sent, then status written.
The send also ran before Slack's 3-second ack, which invites the second click.
**Shipped:** conditional-update claim; ack before work, see [approvals.md](approvals.md).

### 5. Initialization race on cold start
A boolean was set before the awaits. A second webhook during init skipped it and
reached a bot with no handlers registered.
**Shipped:** cached promise, cleared on failure, see [setup.md](setup.md).

### 6. Identity lookup capped at 1000 users, cache never expired
`listUsers({ perPage: 1000 })` filtered in memory; positive matches cached for
the life of the instance, keyed without the workspace.
**Shipped:** single-query contract, `team:user` key, TTL, see
[conversation.md](conversation.md).

### 7. Bot token chosen by unordered `limit(1)`
Tokens were stored per `(user, provider, team)`; several lookups took "the first"
row, one of them with no team filter at all.
**Shipped:** one row per workspace, see [data-model.md](data-model.md).

### 8. Failures were silent
Handler errors were logged and swallowed; refused or failed button clicks
returned 200 with no feedback.
**Shipped:** failure message in thread; ephemeral reply via `response_url`.

### 9. Bots and non-users triggered identity-failure replies
Only the bot's own messages were ignored. Any other bot, and every colleague
without an account in a followed thread, got the "couldn't match" reply.
**Shipped:** bot authors ignored; silent in followed threads.

### 10. Markdown converter corrupted bold headings, bold-italic and arithmetic
Verified by running it: `# **Title**` came out as `**Title**`, `***x***` as
`__x__`, and `2 * 3 * 4` as `2 _ 3 _ 4`.
**Shipped:** placeholder-parking converter with regression tests, see
[outbound.md](outbound.md).

### 11. OAuth return path was an open redirect and broke on query strings
`${appUrl}${returnTo}?flag=1` with `returnTo` read out of `state`.
**Shipped:** `safeReturnPath`, `withParam`, return path in the cookie.

### 12. Signature check could throw instead of rejecting
String-length comparison before a byte-length-sensitive `timingSafeEqual`;
`parseInt` accepted junk timestamps.
**Shipped:** byte-length guard, digit check, tests, see [approvals.md](approvals.md).

### 13. Progress reaction used a custom emoji
`:loading:` exists only where someone uploaded it; elsewhere the call failed
silently. **Shipped:** `hourglass_flowing_sand`.

### 14. Smaller items
Emails and message text in logs; `logger: "debug"` in production; empty-string
defaults for missing team and channel ids; a dual write of installations into
chat state; two exported helpers (`isChannelMonitored`, `resolveTokenForTeam`)
that nothing called.

## Kept deliberately

- **Email as the identity bridge.** No linking step to build or explain. The
  trust decision is spelled out in [adaptation.md](adaptation.md).
- **A separate interactivity route with hand-rolled verification**, rather than
  the SDK's `onAction`. It is what ran in production.
- **`onLockConflict: "force"`**, though deprecated. Same reason; the likely
  successor is named in [setup.md](setup.md).
- **`fullStream` with a plain-text fallback.** Streaming to Slack fails for
  reasons outside the app's control; the fallback is what keeps an answer
  arriving.
- **The eyes reaction stays on the message.** It is the receipt that the bot saw it; only the
  progress emoji is removed.
- **Answering every human message in a followed thread.** A product decision;
  the alternative is described in [conversation.md](conversation.md).
- **Summary as parent, full text in thread** for reports.
- **Six-step tool budget and the four discipline lines in the prompt.**
- **Preview truncated at 1500 characters** inside a code block.

## Added

Not in the earlier implementation. Designed here, and covered by tests where
marked.

- `installationProvider` instead of seeding chat state (type-checked; lookup
  path verified in the adapter's own code; **not run in production**).
- A dedicated `slack_approvals` table and the generic `kind` to executor registry.
  The earlier implementation had one hard-coded approval type, stored inside
  another row's JSON.
- `response_url` ephemeral feedback. (the refusal reasons are tested; the HTTP call is not)
- Single 429 retry honouring `Retry-After`. (tested)
- Differentiated identity failure reasons. (tested)
- Header length guard and `missing_ts` error in the poster.
- All of [operations.md](operations.md) beyond the symptom causes: there was no
  operator surface to carry over.

## Left behind

Channel-history reading, channel listing, web search, semantic search over
resources, third-party data integrations, and per-user execution rate limiting.
The last is worth rebuilding in any host whose actions cost money.

## Fixing a bot that already runs

If you are applying this skill to an existing Slack bot rather than building
one, fix in this order, most damaging first:

1. Add the access check to the two id-only tools (#1).
2. Add the missing scopes to the authorize URL, then have each workspace
   reinstall (#2).
3. Make approve a conditional update and acknowledge before sending (#4).
4. Move chat state to a shared adapter (#3).
5. Cache the init promise (#5).
6. Ignore bot authors; stop replying to non-users in followed threads (#9).
7. Replace the user listing with a lookup by email (#6).
8. Validate the OAuth return path (#11).
9. The rest, in any order.
