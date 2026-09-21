# Setup: Slack app, install flow, bot boot

## File map

```
lib/slack-bot/
  host.ts               every seam; the file you edit      adaptation.md
  services.ts           singletons built from the host     adaptation.md
  types.ts              stores and records                 data-model.md
  stores-supabase.ts    stores-memory.ts                   data-model.md
  oauth.ts              scopes, authorize URL, exchange    this file
  bot.ts                Chat instance, cached init         this file
  identity.ts  context.ts  handlers.ts  tools.ts           conversation.md
  outbound.ts  mrkdwn.ts  strings.ts                       outbound.md
  approvals.ts  verify-signature.ts                        approvals.md
app/api/slack/events/route.ts                              this file
app/api/slack/interactivity/route.ts                       approvals.md
app/api/auth/slack/connect/route.ts                        this file
app/api/auth/slack/callback/route.ts                       this file
```

## Slack app configuration

At api.slack.com/apps, for an app distributed by OAuth:

| Page | Setting | Value |
|---|---|---|
| OAuth & Permissions | Redirect URL | `https://<app>/api/auth/slack/callback` |
| OAuth & Permissions | Bot token scopes | every scope in `SLACK_BOT_SCOPES` below |
| Event Subscriptions | Request URL | `https://<app>/api/slack/events` |
| Event Subscriptions | Bot events | `app_mention`, `message.im`, `message.channels`, `message.groups` |
| Interactivity & Shortcuts | Request URL | `https://<app>/api/slack/interactivity` |
| App Home | Messages tab | on, and "Allow users to send messages" checked, or DMs are disabled |

`message.channels` / `message.groups` are what deliver **thread follow-ups**. With
only `app_mention`, the bot answers the first mention and never sees a reply that
does not mention it again.

Slack verifies the events URL by POSTing a `url_verification` challenge; the
Chat SDK answers it. The route must be deployed, and reachable without a login
redirect, before the page will save.

## Environment

| Variable | Used by |
|---|---|
| `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET` | OAuth install |
| `SLACK_SIGNING_SECRET` | Both webhooks |
| `NEXT_PUBLIC_APP_URL` | Redirect URI. Must match the Slack config byte for byte, including scheme and no trailing slash |
| `AI_GATEWAY_API_KEY` (or Vercel OIDC) | The model |
| `REDIS_URL` or `POSTGRES_URL` | Production chat state |

There is **no `SLACK_BOT_TOKEN`**. A single env token makes the app
single-workspace; tokens come from `slack_installations`, per team.

## Scopes and OAuth helpers

```ts
// lib/slack-bot/oauth.ts
const AUTHORIZE_URL = "https://slack.com/oauth/v2/authorize";
const TOKEN_URL = "https://slack.com/api/oauth.v2.access";

export const OAUTH_STATE_COOKIE = "slack_oauth_state";

/**
 * Bot scopes REQUESTED at install time. Slack grants what the authorize URL
 * asks for, not what the app config page lists, so a scope missing here is
 * missing from every token this flow produces.
 */
export const SLACK_BOT_SCOPES = [
  "app_mentions:read", // app_mention events
  "chat:write", // reply, post reports, rewrite approval messages
  "im:history", // message.im events: DMs to the bot
  "im:read",
  "im:write",
  "channels:history", // follow-ups in public-channel threads
  "groups:history", // follow-ups in private-channel threads
  "channels:read", // conversations.info for get_channel_info
  "groups:read",
  "reactions:write", // the eyes and hourglass receipts
  "users:read",
  "users:read.email", // identity. Without it nobody can ever be matched.
] as const;

function redirectUri(appUrl: string): string {
  return `${appUrl.replace(/\/$/, "")}/api/auth/slack/callback`;
}

export function buildAuthorizeUrl(input: {
  clientId: string;
  appUrl: string;
  state: string;
}): string {
  const url = new URL(AUTHORIZE_URL);
  url.searchParams.set("client_id", input.clientId);
  url.searchParams.set("scope", SLACK_BOT_SCOPES.join(","));
  url.searchParams.set("redirect_uri", redirectUri(input.appUrl));
  url.searchParams.set("state", input.state);
  return url.toString();
}

/**
 * Only same-site paths survive. `//evil.com` and `/\evil.com` are protocol-relative
 * URLs, and `@evil.com` appended to the app URL turns the app host into a username.
 */
export function safeReturnPath(value: string | null | undefined): string {
  if (!value || !value.startsWith("/") || value.startsWith("//") || value.includes("\\")) {
    return "/";
  }
  return value;
}

/** Appends a query parameter without breaking a return path that already has a query. */
export function withParam(appUrl: string, path: string, key: string, value: string): string {
  const url = new URL(safeReturnPath(path), appUrl);
  url.searchParams.set(key, value);
  return url.toString();
}

export interface SlackOAuthGrant {
  botToken: string;
  botUserId: string;
  teamId: string;
  teamName: string;
}

export async function exchangeCode(
  input: { clientId: string; clientSecret: string; appUrl: string; code: string },
  fetchImpl: typeof fetch = fetch,
): Promise<SlackOAuthGrant> {
  const res = await fetchImpl(TOKEN_URL, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      client_id: input.clientId,
      client_secret: input.clientSecret,
      code: input.code,
      // Must equal the redirect_uri of the authorize step byte for byte.
      redirect_uri: redirectUri(input.appUrl),
    }),
  });
  if (!res.ok) throw new Error(`Slack token exchange HTTP ${res.status}`);

  const data = (await res.json()) as {
    ok: boolean;
    error?: string;
    access_token?: string;
    bot_user_id?: string;
    team?: { id?: string; name?: string };
  };
  if (!data.ok || !data.access_token || !data.bot_user_id || !data.team?.id) {
    throw new Error(`Slack token exchange failed: ${data.error ?? "missing fields"}`);
  }
  return {
    botToken: data.access_token,
    botUserId: data.bot_user_id,
    teamId: data.team.id,
    teamName: data.team.name ?? "",
  };
}
```

The scope list is the defect most worth internalizing. The earlier
implementation listed `users:read.email` and `app_mentions:read` in its setup
documentation and on the Slack config page, but its authorize URL asked for
neither. Slack issues tokens
with the scopes **the URL requests**. Every workspace installed through the
app's own Connect button would get a token that cannot read emails, and every
person would be told their account could not be matched. Installing from the
Slack config page's own "Install to Workspace" button grants the full list,
which is how this kind of bug hides during development.

After changing scopes, existing workspaces keep their old token until someone
reinstalls. There is no way to upgrade a token in place.

## Install routes

```ts
// app/api/auth/slack/connect/route.ts
import { randomBytes } from "node:crypto";

import { cookies } from "next/headers";
import { NextResponse } from "next/server";

import { botHost } from "@/lib/slack-bot/host";
import { buildAuthorizeUrl, OAUTH_STATE_COOKIE, safeReturnPath } from "@/lib/slack-bot/oauth";

export async function GET(request: Request): Promise<Response> {
  const userId = await botHost.getCurrentUserId();
  if (!userId) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const clientId = process.env.SLACK_CLIENT_ID;
  const appUrl = process.env.NEXT_PUBLIC_APP_URL;
  if (!clientId || !appUrl) {
    return NextResponse.json({ error: "Slack is not configured" }, { status: 500 });
  }

  const nonce = randomBytes(16).toString("hex");
  const returnTo = safeReturnPath(new URL(request.url).searchParams.get("return_to"));

  // The return path rides in the cookie, not in `state`: `state` round-trips
  // through Slack and the browser's address bar, the cookie does not.
  const cookieStore = await cookies();
  cookieStore.set(OAUTH_STATE_COOKIE, JSON.stringify({ nonce, returnTo }), {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax", // "strict" would drop the cookie on the redirect back from slack.com
    maxAge: 600,
    path: "/",
  });

  return NextResponse.redirect(buildAuthorizeUrl({ clientId, appUrl, state: nonce }));
}
```

```ts
// app/api/auth/slack/callback/route.ts
import { timingSafeEqual } from "node:crypto";

import { cookies } from "next/headers";
import { NextResponse } from "next/server";

import { botHost } from "@/lib/slack-bot/host";
import { exchangeCode, OAUTH_STATE_COOKIE, withParam } from "@/lib/slack-bot/oauth";

function sameNonce(a: string, b: string): boolean {
  const x = Buffer.from(a, "utf8");
  const y = Buffer.from(b, "utf8");
  return x.length === y.length && timingSafeEqual(x, y);
}

export async function GET(request: Request): Promise<Response> {
  const appUrl = process.env.NEXT_PUBLIC_APP_URL ?? new URL(request.url).origin;
  const params = new URL(request.url).searchParams;
  const fail = (path: string, code: string): Response =>
    NextResponse.redirect(withParam(appUrl, path, "slack_error", code));

  const cookieStore = await cookies();
  const saved = cookieStore.get(OAUTH_STATE_COOKIE)?.value;
  cookieStore.delete(OAUTH_STATE_COOKIE); // single use, whatever happens next

  let nonce = "";
  let returnTo = "/";
  try {
    const parsed = JSON.parse(saved ?? "") as { nonce?: string; returnTo?: string };
    nonce = parsed.nonce ?? "";
    returnTo = parsed.returnTo ?? "/";
  } catch {
    return fail("/", "invalid_state");
  }

  const state = params.get("state") ?? "";
  if (!nonce || !sameNonce(nonce, state)) return fail("/", "invalid_state");
  if (params.get("error")) return fail(returnTo, params.get("error") ?? "access_denied");

  const code = params.get("code");
  if (!code) return fail(returnTo, "missing_code");

  const userId = await botHost.getCurrentUserId();
  if (!userId) return fail("/", "signed_out");

  try {
    const grant = await exchangeCode({
      clientId: process.env.SLACK_CLIENT_ID ?? "",
      clientSecret: process.env.SLACK_CLIENT_SECRET ?? "",
      appUrl,
      code,
    });
    // One write. The adapter reads this table per webhook, so there is no
    // second copy in chat state to fall out of step.
    await botHost.installations.upsert({
      teamId: grant.teamId,
      teamName: grant.teamName,
      botToken: grant.botToken,
      botUserId: grant.botUserId,
      installedByUserId: userId,
    });
  } catch (err) {
    console.error("[slack-bot] install failed", err);
    return fail(returnTo, "install_failed");
  }

  return NextResponse.redirect(withParam(appUrl, returnTo, "slack_connected", "1"));
}
```

Decisions in these two files:

- **The return path lives in the cookie.** The earlier implementation packed it
  into `state` as `nonce:encodedPath` and redirected to `${appUrl}${returnTo}`.
  A path of `@evil.com` makes that `https://app.example@evil.com`: the app's
  host becomes a username and the browser goes to `evil.com`. `safeReturnPath`
  plus `new URL(path, appUrl)` closes it twice over.
- **`withParam` instead of string concatenation.** `${returnTo}?slack_connected=1`
  produces `/settings?tab=slack?slack_connected=1` when the path already has a
  query, and the app never sees the flag.
- **The state cookie is deleted before anything can fail**, so a nonce is never
  reusable.
- **The user declining** (`?error=access_denied`) is handled after the state
  check, so an attacker cannot use the error branch as an unauthenticated
  redirect either.

## Chat state

The state adapter holds three things the bot cannot work without across
instances:

| Held in state | With memory state on serverless |
|---|---|
| Thread subscriptions | A follow-up that lands on a fresh instance is not recognized as a followed thread. The bot answered once and then goes quiet, intermittently |
| Dedupe keys | Slack's retry of a slow event lands on another instance and is answered a second time |
| Thread locks | Two instances stream into the same thread at once |

The earlier implementation ran memory state on serverless; the first row is the
symptom its users saw. **Use memory state only for local development.**

In `host.ts`, swap `createChatState`. *These two variants follow the adapters'
own documentation and were not compiled by this skill's verification, because
the packages were not installed where it ran.*

```ts
import { createRedisState } from "@chat-adapter/state-redis";
createChatState: () => createRedisState(),      // reads REDIS_URL
```

```ts
import { createPostgresState } from "@chat-adapter/state-pg";
createChatState: () => createPostgresState(),   // reads POSTGRES_URL or DATABASE_URL
```

Postgres is the no-new-infrastructure choice when the app already has a
database; Redis is lower latency for the lock on every message.

## The bot

```ts
// lib/slack-bot/bot.ts
import { createSlackAdapter } from "@chat-adapter/slack";
import { Chat } from "chat";

import { registerHandlers } from "./handlers";
import { botHost } from "./host";

export interface BotRuntime {
  bot: Chat;
}

function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is not configured`);
  return value;
}

async function init(): Promise<BotRuntime> {
  const slack = createSlackAdapter({
    clientId: requireEnv("SLACK_CLIENT_ID"),
    clientSecret: requireEnv("SLACK_CLIENT_SECRET"),
    signingSecret: requireEnv("SLACK_SIGNING_SECRET"),
    // The app's own table is the single source of truth for bot tokens. The
    // adapter asks per webhook, so a workspace installed a second ago works on
    // every instance and nothing has to be copied into chat state at boot.
    installationProvider: {
      getInstallation: async (installationId) => {
        const row = await botHost.installations.get(installationId);
        if (!row) return null;
        return {
          botToken: row.botToken,
          botUserId: row.botUserId ?? undefined,
          teamName: row.teamName ?? undefined,
        };
      },
    },
  });

  const bot = new Chat({
    userName: botHost.botName,
    adapters: { slack },
    state: botHost.createChatState(),
    // Slack rate-limits message edits. 800ms stays smooth without hitting the limit.
    streamingUpdateIntervalMs: 800,
    // An AI reply holds the thread lock for many seconds. The default ("drop")
    // silently discards a follow-up sent meanwhile; "force" answers it, at the
    // cost of two replies briefly streaming at once.
    onLockConflict: "force",
    logger: process.env.NODE_ENV === "production" ? "warn" : "debug",
  });

  registerHandlers(bot);
  await bot.initialize();
  return { bot };
}

let ready: Promise<BotRuntime> | null = null;

/**
 * Cache the PROMISE, not a boolean. With a flag set before the awaits, a second
 * webhook arriving during a cold start skips init and reaches a bot with no
 * handlers registered, and the event is dropped without an error.
 */
export function getBot(): Promise<BotRuntime> {
  ready ??= init().catch((err: unknown) => {
    ready = null; // let the next request retry instead of caching the failure
    throw err;
  });
  return ready;
}
```

- **`installationProvider`** makes `slack_installations` the only place a token
  lives. The earlier implementation instead copied every token into chat state
  at cold start (`setInstallation` in a loop) and again in the OAuth callback,
  with a
  comment noting the second write could fail and would "be picked up on next
  cold start". With memory state that also meant a workspace installed on
  instance A did not exist on instance B. *The provider option is present in
  `@chat-adapter/slack` 4.40+ and its lookup path was verified in the adapter's
  own code, but this configuration has not run in production; the seeding
  approach has.* If you must seed instead, do it inside `init()` so it is
  covered by the cached promise, and use a shared state adapter.
- **`onLockConflict: "force"`** is marked deprecated in favour of the
  `concurrency` option (`"queue"`, `"debounce"`, ...) in current SDK versions. It
  is kept because it is what the earlier implementation ran.
  `concurrency: "queue"` is the
  likely better answer, since it serializes replies instead of overlapping them;
  try it, and watch for follow-ups being answered late rather than dropped.

## Events route

```ts
// app/api/slack/events/route.ts
import { after } from "next/server";

import { getBot } from "@/lib/slack-bot/bot";

// Slack retries anything not acknowledged within 3 seconds. The SDK verifies the
// signature, answers at once, and hands the real work to `waitUntil`; `after`
// keeps the function alive until that work settles.
export const maxDuration = 300;

export async function POST(request: Request): Promise<Response> {
  const { bot } = await getBot();
  // `webhooks` is keyed by adapter name, so it is possibly undefined under
  // noUncheckedIndexedAccess. It exists whenever the slack adapter is registered.
  const handleSlack = bot.webhooks.slack;
  if (!handleSlack) return new Response("Slack adapter not registered", { status: 500 });
  return handleSlack(request, {
    waitUntil: (task) => after(() => task),
  });
}
```

The SDK verifies the signature, answers the challenge, drops duplicates and
returns 200 immediately; handlers run inside the `waitUntil` task. Without
`after` (or `waitUntil` from `@vercel/functions`) the platform may freeze the
function as soon as the response is sent and the reply is cut off mid-stream.

Do not set `runtime = "edge"`. The handlers need Node APIs and minutes, not
seconds.

## Checklist

- [ ] All four bot events subscribed, including `message.channels` and `message.groups`
- [ ] Messages tab enabled in App Home
- [ ] Authorize URL scope list equals the config page's list
- [ ] Redirect URL identical in Slack config, authorize step and token exchange
- [ ] Webhook routes reachable without a session (no proxy redirect)
- [ ] Shared chat state adapter in production
- [ ] `getBot()` caches the promise and clears it on failure
- [ ] `maxDuration` raised on both webhook routes
