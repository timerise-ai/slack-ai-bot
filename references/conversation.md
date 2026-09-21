# Conversation: identity, handlers, tools

The inbound half. A message arrives, the author is resolved to an app user, and
a model answers with tools bound to that user.

## Flow

```
event ─▶ author is me or a bot? ─ yes ─▶ ignore
           │ no
           ▼
        origin = { team, channel, thread }   (null → log, stop)
           ▼
        token  = slack_installations[team]   (none → log, stop)
           ▼
        identity: users.info → email → findUserIdByEmail
           │ fail: tell the person, unless this is a followed thread
           ▼
        👀 + ⏳ ─▶ subscribe (first contact only) ─▶ streamText(tools(userId))
           │ throw → post a failure message
           ▼
        remove ⏳   (👀 stays as the "seen" receipt)
```

## Identity

```ts
// lib/slack-bot/identity.ts
export type IdentityResult =
  | { ok: true; userId: string }
  | {
      ok: false;
      reason: "slack_api_error" | "no_email" | "no_matching_user";
      detail?: string;
    };

export type SlackUserResolver = (
  botToken: string,
  teamId: string,
  slackUserId: string,
) => Promise<IdentityResult>;

interface ResolverDeps {
  /** Host seam: ONE indexed lookup by lower-cased email. Never list users and filter. */
  findUserIdByEmail: (email: string) => Promise<string | null>;
  fetchImpl?: typeof fetch;
  /** How long a positive match is trusted. Bounds how long a removed user keeps access. */
  ttlMs?: number;
  now?: () => number;
}

/**
 * Maps a Slack member to an app user through the Slack profile email.
 * Requires the `users:read.email` bot scope; without it Slack omits the email
 * and every caller fails with `no_email`.
 */
export function createSlackUserResolver(deps: ResolverDeps): SlackUserResolver {
  const fetchImpl = deps.fetchImpl ?? fetch;
  const ttlMs = deps.ttlMs ?? 10 * 60 * 1000;
  const now = deps.now ?? Date.now;
  const cache = new Map<string, { userId: string; expiresAt: number }>();

  return async (botToken, teamId, slackUserId) => {
    // Slack user ids are only unique inside a workspace.
    const key = `${teamId}:${slackUserId}`;
    const hit = cache.get(key);
    if (hit && hit.expiresAt > now()) return { ok: true, userId: hit.userId };
    cache.delete(key);

    let data: {
      ok: boolean;
      error?: string;
      user?: { profile?: { email?: string } };
    };
    try {
      const res = await fetchImpl(
        `https://slack.com/api/users.info?user=${encodeURIComponent(slackUserId)}`,
        { headers: { Authorization: `Bearer ${botToken}` } },
      );
      data = (await res.json()) as typeof data;
    } catch (err) {
      return { ok: false, reason: "slack_api_error", detail: String(err) };
    }
    if (!data.ok) {
      return { ok: false, reason: "slack_api_error", detail: data.error };
    }

    const email = data.user?.profile?.email?.trim().toLowerCase();
    if (!email) return { ok: false, reason: "no_email" };

    const userId = await deps.findUserIdByEmail(email);
    // Misses are not cached: someone who signs up a minute later must get in.
    if (!userId) return { ok: false, reason: "no_matching_user" };

    cache.set(key, { userId, expiresAt: now() + ttlMs });
    return { ok: true, userId };
  };
}
```

What changed from the source, and why each matters:

| Source behaviour | Consequence | Here |
|---|---|---|
| Listed up to 1000 auth users, filtered in memory | User 1001 can never be matched; a full user listing on every uncached message | `findUserIdByEmail`, one query |
| Cache keyed by Slack user id, never expired | A user removed from the app keeps bot access until the instance dies; ids collide across workspaces | Keyed `team:user`, 10-minute TTL |
| One failure message for every cause | "Use the same email" shown when the real cause is a missing scope, which no user can fix | Three reasons, three messages |
| Logged every resolved email address | Personal data in logs for no operational gain | Logs the reason only |

## Origin

```ts
// lib/slack-bot/context.ts
import type { SlackOrigin } from "./types";

/**
 * Reads workspace, channel and thread out of a raw Slack message event.
 * Returns null when team or channel is missing: an origin with an empty id
 * posts nowhere and fails far away from the cause.
 */
export function extractSlackOrigin(raw: unknown): SlackOrigin | null {
  if (!raw || typeof raw !== "object") return null;
  const event = raw as Record<string, unknown>;
  const str = (value: unknown): string | null =>
    typeof value === "string" && value.length > 0 ? value : null;

  // `team` on message events, `team_id` on some wrappers and app_mention payloads.
  const teamId = str(event.team) ?? str(event.team_id);
  const channelId = str(event.channel);
  if (!teamId || !channelId) return null;

  // A reply carries thread_ts. A top-level message IS the thread root, so its own ts is.
  return { teamId, channelId, threadTs: str(event.thread_ts) ?? str(event.ts) };
}
```

The source defaulted missing ids to `""` and carried on. An empty `teamId` then
failed inside the adapter's installation lookup, and an empty `channelId` failed
later inside a tool, both far from the cause.

## Handlers

```ts
// lib/slack-bot/handlers.ts
import { stepCountIs, streamText } from "ai";
import { type Chat, type Message, type Thread, toAiMessages } from "chat";

import { extractSlackOrigin } from "./context";
import { botHost } from "./host";
import type { IdentityResult } from "./identity";
import { approvalService, resolveSlackUser } from "./services";
import { BOT_STRINGS } from "./strings";
import { createBotTools } from "./tools";
import type { SlackOrigin } from "./types";

type Trigger = "mention" | "dm" | "subscribed";

/** Reactions are receipts, never load-bearing: a missing scope must not block an answer. */
async function react(
  op: "add" | "remove",
  thread: Thread,
  message: Message,
  emoji: string,
): Promise<void> {
  const call =
    op === "add"
      ? thread.adapter.addReaction(thread.id, message.id, emoji)
      : thread.adapter.removeReaction(thread.id, message.id, emoji);
  await call.catch(() => {});
}

function identityFailureText(result: Extract<IdentityResult, { ok: false }>): string {
  if (result.reason === "no_email") return BOT_STRINGS.identityNoEmail;
  if (result.reason === "no_matching_user") return BOT_STRINGS.identityNoMatch;
  return BOT_STRINGS.identityUnavailable;
}

async function answer(
  thread: Thread,
  userId: string,
  origin: SlackOrigin,
  botToken: string,
): Promise<void> {
  // Pull in replies that arrived since the webhook payload was built.
  await thread.refresh();
  const history = await toAiMessages(thread.recentMessages, { includeNames: true });

  const result = streamText({
    model: botHost.model,
    system: botHost.systemPrompt,
    messages: history,
    tools: createBotTools(botHost, approvalService, userId, { origin, botToken }),
    // A hard ceiling on tool loops. The prompt asks for restraint; this enforces it.
    stopWhen: stepCountIs(6),
  });

  try {
    // fullStream, not textStream: step boundaries let the adapter close one
    // Slack message segment per step instead of gluing tool chatter together.
    await thread.post(result.fullStream);
  } catch (err) {
    console.error("[slack-bot] streamed post failed, falling back to plain text", err);
    const text = await Promise.resolve(result.text).catch(() => "");
    await thread.post(text.trim() || BOT_STRINGS.streamFallback);
  }
}

async function handle(trigger: Trigger, thread: Thread, message: Message): Promise<void> {
  // Bots have no email, so without this every bot message in a followed thread
  // gets an "I can't identify you" reply, and two bots can answer each other forever.
  if (message.author.isMe || message.author.isBot === true) return;

  const origin = extractSlackOrigin(message.raw);
  if (!origin) {
    console.error("[slack-bot] message without team or channel", { threadId: thread.id });
    return;
  }
  const installation = await botHost.installations.get(origin.teamId);
  if (!installation) {
    console.error("[slack-bot] no installation for team", origin.teamId);
    return;
  }
  const { botToken } = installation;

  const identity = await resolveSlackUser(botToken, origin.teamId, message.author.userId);
  if (!identity.ok) {
    console.warn("[slack-bot] identity failed", identity.reason, identity.detail ?? "");
    // In a followed thread, colleagues without an account are just talking to
    // each other. Only tell people who addressed the bot directly.
    if (trigger !== "subscribed") await thread.post(identityFailureText(identity));
    return;
  }

  await react("add", thread, message, "eyes");
  await react("add", thread, message, "hourglass_flowing_sand");
  try {
    if (trigger !== "subscribed") await thread.subscribe();
    await answer(thread, identity.userId, origin, botToken);
  } catch (err) {
    console.error(`[slack-bot] ${trigger} handler failed`, err);
    // Silence after the eyes reaction reads as "still working". Say it failed.
    await thread.post(BOT_STRINGS.answerFailed).catch(() => {});
  } finally {
    await react("remove", thread, message, "hourglass_flowing_sand");
  }
}

export function registerHandlers(bot: Chat): void {
  bot.onNewMention((thread, message) => handle("mention", thread, message));
  bot.onDirectMessage((thread, message) => handle("dm", thread, message));
  bot.onSubscribedMessage((thread, message) => handle("subscribed", thread, message));
}
```

Reasoning behind the non-obvious lines:

- **`isBot === true`**, not truthiness: the SDK reports `"unknown"` when the
  platform did not say, and an unknown author should still be answered.
- **Silent identity failure in followed threads.** Once the bot follows a
  thread, it sees every reply. The source answered each reply from a colleague
  without an account with "I couldn't match your Slack account", turning a team
  discussion into a wall of bot apologies.
- **Identity before reactions.** Reacting first puts 👀 on messages the bot then
  ignores.
- **`hourglass_flowing_sand`, not a custom emoji.** The source used `:loading:`,
  which exists only in workspaces that uploaded it. The call failed silently
  everywhere else, so most users never saw a progress indicator.
- **The catch posts.** The source logged and returned; the user saw 👀, then
  nothing, forever.
- **`thread.subscribe()` only on first contact.** It is a state write; repeating
  it on every follow-up is wasted latency.
- **`stepCountIs(6)`** is a budget, not a suggestion. Without it a model that
  keeps calling tools runs until the function times out, holding the thread lock.

The bot answers **every** human message in a followed thread, mention or not.
That is the source's behaviour and suits DM-like threads. In busy channels it is
noisy; the alternative is to return early from the `"subscribed"` trigger unless
`message.isMention` is set, and to say so in the first reply.

## Tools

```ts
// lib/slack-bot/tools.ts
import { tool } from "ai";
import { z } from "zod";

import type { ApprovalService } from "./approvals";
import type { BotHost } from "./host";
import { slackApi } from "./outbound";
import type { SlackOrigin } from "./types";

/** Tool results go back into the prompt. Cap them or one large record eats the context. */
const RESULT_MAX = 8000;

export interface ToolContext {
  origin: SlackOrigin | null;
  botToken: string;
}

/**
 * Tools for ONE resolved user. `userId` is closed over, never a tool input:
 * a model can be talked into passing someone else's id, a closure cannot.
 */
export function createBotTools(
  host: BotHost,
  approvals: ApprovalService,
  userId: string,
  ctx: ToolContext,
) {
  return {
    search_resources: tool({
      description:
        "Search the user's resources by name. Use this to find an id whenever the user refers to something by name.",
      inputSchema: z.object({ query: z.string().describe("Text to match against names.") }),
      execute: async ({ query }) => {
        const results = await host.searchResources(userId, query);
        return results.length > 0 ? results : "Nothing matched.";
      },
    }),

    get_resource: tool({
      description: "Get the latest content of one resource by id.",
      inputSchema: z.object({ resource_id: z.string().describe("Id from search_resources.") }),
      execute: async ({ resource_id }) => {
        const resource = await host.getResource(userId, resource_id);
        // Same answer for "missing" and "not yours": do not confirm that an id exists.
        if (!resource) return "Not found, or the user has no access.";
        return { ...resource, content: resource.content.slice(0, RESULT_MAX) };
      },
    }),

    run_action: tool({
      description:
        "Run a resource's action now. Use when the user asks to run, send, publish or create something. Pass the user's own instructions as `input`.",
      inputSchema: z.object({
        resource_id: z.string().describe("Id from search_resources."),
        input: z.string().optional().describe("The user's instructions, verbatim."),
      }),
      execute: ({ resource_id, input }) =>
        host.runAction(userId, resource_id, input, {
          origin: ctx.origin,
          requestApproval: async (approval, preview) => {
            if (!ctx.origin) throw new Error("No Slack thread to ask for approval in");
            await approvals.request(
              { ...approval, origin: ctx.origin, requestedByUserId: userId },
              preview,
            );
          },
        }),
    }),

    get_channel_info: tool({
      description:
        "Get the current Slack channel's name, topic and purpose. Use it to pick between similar resources when the channel hints at the right one.",
      inputSchema: z.object({}),
      execute: async () => {
        if (!ctx.origin) return "Channel context not available.";
        try {
          const data = await slackApi(ctx.botToken, "conversations.info", {
            channel: ctx.origin.channelId,
          });
          const channel = data.channel as
            | { name?: string; topic?: { value?: string }; purpose?: { value?: string } }
            | undefined;
          return {
            name: channel?.name ?? "",
            topic: channel?.topic?.value ?? "",
            purpose: channel?.purpose?.value ?? "",
          };
        } catch {
          return "Could not fetch channel info.";
        }
      },
    }),
  };
}
```

**Every tool is an authorization boundary.** The source app's `get_tile_status`
and `get_tile_result` took an id and queried with the service-role client, with
no check that the asking user could see that tile. Any matched Slack user who
learned or guessed an id could read another customer's results by asking the bot
for them. Its `run_tile` did check. The pattern that prevents the mismatch is
structural: **`BotHost` has no method that takes an id without a `userId`**, so
an unscoped lookup cannot be written by accident.

Rules for adding tools:

- Inputs come from the model, which takes them from anyone in the thread.
  Validate with zod, then authorize in the host method.
- Return small results. `RESULT_MAX` caps content; a list tool should cap count.
- Descriptions are prompt text: say **when** to call the tool, not just what it
  does. The source's descriptions all carry a "use this when…" clause.
- A tool that causes an irreversible effect calls `requestApproval` instead of
  acting — [approvals.md](approvals.md).
- One write per request. The system prompt's "call run_action AT MOST ONCE" is
  carried over from the source, whose prompt had a dedicated tool-use discipline
  section. A model that retries a write with different parameters after an
  ambiguous result creates duplicates, and nothing downstream can tell.

## System prompt

The prompt in `host.ts` is deliberately short. Four lines are carried over from
the source app's prompt, where they formed an explicit discipline section. What
each guards against:

| Line | Guards against |
|---|---|
| "Always look data up with the tools. Never guess ids" | Invented UUIDs |
| "call search_resources to get its id first" | Passing a name where an id is expected |
| "run_action AT MOST ONCE … do not retry with different parameters" | Duplicate side effects |
| "After every tool call either answer in text or make exactly one more tool call" | Runs that end at the step limit having said nothing |

Add domain vocabulary above them; keep them.

## Checklist

- [ ] Bot and self messages ignored before any work
- [ ] No tool accepts `userId`; no host method takes an id without one
- [ ] Identity failures differentiated; silent in followed threads
- [ ] Failure path posts a message
- [ ] Progress emoji is a standard one
- [ ] Step budget set
