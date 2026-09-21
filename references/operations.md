# Operations

What an operator needs to see and do. In the earlier implementation the only
operator surface was the function log, so every symptom below was diagnosed by
reading a log stream after someone complained.

## Symptom table

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| Nobody can be identified ("can't read your Slack email") | Token lacks `users:read.email` | `auth.test` response header `x-oauth-scopes` for that token | Correct `SLACK_BOT_SCOPES`, reinstall the workspace |
| One person can't be identified | Slack email is not the app email | Compare the two | Align emails, or add account linking |
| Answers the mention, ignores thread replies | `message.channels` / `message.groups` not subscribed | Slack app, Event Subscriptions | Subscribe; reinstall if scopes changed |
| Ignores thread replies **sometimes** | Memory chat state across instances | `createChatState` in `host.ts` | Shared adapter, see [setup.md](setup.md) |
| Answers twice | Memory state (no shared dedupe), or a handler slower than the dedupe TTL | Same; and `dedupeTtlMs` | Shared adapter; raise `dedupeTtlMs` |
| The eyes reaction, then nothing | Handler threw before posting; or the function was frozen after the ack | Logs for `handler failed`; is `after`/`waitUntil` wired, `maxDuration` raised | Fix the throw; wire `after` |
| Slack: "Your URL didn't respond" when saving the events URL | Auth proxy redirecting the webhook, or route not deployed | `curl -i -X POST https://<app>/api/slack/events` should be 401, not 302 | Exclude the routes from the proxy |
| Button shows "operation timed out" | Work running before the ack | Interactivity route returns before `after()`? | Ack first, see [approvals.md](approvals.md) |
| Click does nothing | Refusal with no `response_url` reply, or unknown `action_id` | Logs; payload's `action_id` | Reply ephemerally; register the id |
| Approval stuck in `processing` | Function died mid-executor | Query below | Decide by hand; see below |
| `not_in_channel` on reports | Bot not invited | n/a | `/invite @bot`, or `chat:write.public` |
| `token_revoked` / `account_inactive` | App uninstalled | n/a | Remove the row; prompt reconnect |
| DMs impossible ("Sending messages to this app has been turned off") | Messages tab disabled | App Home settings | Enable it |
| Everything 401s after a deploy | `SLACK_SIGNING_SECRET` missing in that environment | Env | Set it |

## Stuck approvals

`processing` means an executor started and nobody recorded how it ended. The
side effect may or may not have happened; that is why nothing retries it
automatically.

```sql
select id, kind, resource_id, decided_by_user_id, decided_at
from slack_approvals
where status = 'processing' and decided_at < now() - interval '10 minutes';
```

Check the executor's own system (was the email sent?), then set the row to
`approved` or `failed` by hand. If executors are idempotent on `approval.id`
(pass it as the idempotency key to the email or payment provider), a sweep may
safely re-run them; *that sweep is not part of this skill.*

## Uninstalls

When a workspace removes the app, Slack sends `app_uninstalled` and
`tokens_revoked` events and the token dies. *The earlier implementation handled
neither, and these templates ship no handler either:* the row stays, and
every later call fails with `token_revoked`. Minimum handling:

- Subscribe to both events; on receipt call `botHost.installations.remove(teamId)`.
- In outbound callers, treat `SlackApiError` with code `token_revoked`,
  `account_inactive` or `invalid_auth` as "disconnected": remove the row and
  surface "Reconnect Slack" in the app's settings.

## What to make visible

If the app has an admin or settings surface, these are cheap and each one
replaces a log-reading session:

| Show | From |
|---|---|
| Connected workspaces: name, installed by, when | `slack_installations` |
| Granted scopes vs required, per workspace | `x-oauth-scopes` from `auth.test`, against `SLACK_BOT_SCOPES` |
| Last outbound delivery error per destination | Record it where `postReport` is called |
| Open approvals older than a day | The partial index on `slack_approvals` |
| Disconnect button | `installations.remove` plus `auth.revoke` |

None of these ran in the earlier implementation. They are designs, and they are
marked as such in [provenance.md](provenance.md).

## Logging

Log `teamId`, `channelId`, the trigger, tool names and finish reasons. Do not log
message text, email addresses or tokens. The earlier implementation logged a
resolved email address on every message, and the opening of the message with it.

`logger: "debug"` on the Chat instance prints every webhook payload. The template
restricts it to non-production.

## Cost and abuse

Every message in a followed thread is a model call with tools. The earlier
implementation rate-limited action **execution** per user but not conversation.
If the bot is in
large channels, add a per-user message budget in `handle()` before `answer()`,
and consider replying only to mentions in followed threads, see
[conversation.md](conversation.md).

## Go-live checklist

- [ ] Manual end-to-end script passed, see [testing.md](testing.md)
- [ ] Shared chat state adapter configured in production
- [ ] Installed via the app's own Connect flow; consent screen showed email access
- [ ] Webhook routes return 401 (not 302, not 500) to an unsigned POST
- [ ] Stuck-approval query saved somewhere an operator will find it
- [ ] Uninstall handling decided
- [ ] No message text, emails or tokens in logs
- [ ] `bot_token` storage decision recorded, see [data-model.md](data-model.md)
