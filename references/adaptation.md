# Adaptation

Everywhere this module touches its host, as one table and one file. Fill the
right-hand column before writing anything else.

## The Adaptation Contract

| Seam | The skill ships | The host supplies |
|---|---|---|
| **Domain entities** | `Resource` (something the user can ask about) and `Action` (something the user can run) | Its own nouns; see the rename table |
| **Tenant scope** | `userId`, resolved server-side from the Slack email, closed over by every tool | Its user id, plus whatever org / workspace check `canAccess` needs |
| **Auth guard** | `BotHost.getCurrentUserId()` for the OAuth routes; Slack's signature for webhooks | Clerk, NextAuth, Supabase, custom session |
| **Identity bridge** | `findUserIdByEmail(email)` contract: one indexed lookup | A query against its users table |
| **Data access** | `InstallationStore` and `ApprovalStore` interfaces, Supabase and in-memory implementations | Its ORM or SDK; mapping notes in [data-model.md](data-model.md) |
| **Chat state** | `BotHost.createChatState()` | Redis or Postgres state adapter in production |
| **Model** | `BotHost.model`, an AI Gateway `"provider/model"` string | Its model, provider options, system prompt |
| **Side effects** | `approvalExecutors[kind]` signature | Email send, publish, payment, whatever waits for a human |
| **Background work** | "ack now, work after": `after()` from `next/server` | `waitUntil` from `@vercel/functions`, or a queue |
| **Object storage** | None | None |
| **UI primitives** | Block Kit structure only: section, divider, preview, actions | Nothing; Slack renders it. A "Connect Slack" button in the host's own UI kit links to `/api/auth/slack/connect?return_to=…` |
| **Styling** | None | None |
| **Strings** | `BOT_STRINGS`, one block, keys not inline literals | Translations; a per-locale lookup if the product is multilingual |
| **Validation** | zod schemas on tool inputs | Already zod via the AI SDK; nothing to swap |
| **Tests** | 31 tests on the pure logic, `bun:test` | One import line changes for vitest |

## Rename table

Decide once, apply everywhere: types, tool names, tool descriptions, the system
prompt, table columns. Tool names and descriptions are **prompt text**, so a
half-renamed tool set makes the model call the wrong thing.

| Canonical | Meaning | Source app used | Your app |
|---|---|---|---|
| `Resource` | A thing the user owns and asks about | Tile (inside a Mosaic) | ← confirm |
| `resourceId` | What access is checked against | tile id → mosaic membership | |
| `search_resources` | Find by name | `search`, `find_tile` (vector search) | |
| `get_resource` | Latest content | `get_tile_result`, `get_tile_status` | |
| `run_action` | Execute now | `run_tile` | |
| `Approval` | A parked side effect | Offer draft | |
| `kind` | Which executor | `offer_sender` only | |

Do not rename Slack's own terms: `team_id`, `channel`, `thread_ts`, `ts`,
`action_id`, `block_actions`, `response_url` belong to the platform.

## The host file

One file holds every seam. It ships with a demo implementation so the bot
answers on day one; **replace every body under the divider**. The interface
above it is the contract.

```ts
// lib/slack-bot/host.ts
import { createMemoryState } from "@chat-adapter/state-memory";
import type { StateAdapter } from "chat";

import type { ApprovalExecutor } from "./approvals";
import { createMemoryApprovalStore, createMemoryInstallationStore } from "./stores-memory";
import type { ApprovalStore, InstallationStore, NewApproval, SlackOrigin } from "./types";

export interface ResourceSummary {
  id: string;
  name: string;
  kind: string;
}

export interface ResourceDetail extends ResourceSummary {
  /** Latest state the model may quote. Trim it: this goes into the prompt. */
  content: string;
  updatedAt: string | null;
}

export interface ActionContext {
  /** Null when the action was not started from Slack. */
  origin: SlackOrigin | null;
  /** Parks a side effect behind Approve / Cancel buttons in the originating thread. */
  requestApproval(
    approval: Pick<NewApproval, "kind" | "resourceId" | "summary" | "payload">,
    preview: string,
  ): Promise<void>;
}

export interface ActionOutcome {
  success: boolean;
  /** One sentence the model relays to the user. */
  message: string;
  content?: string;
}

/**
 * Everything the bot needs from the app it lives in. Every domain method takes
 * the server-resolved `userId` and MUST scope by it: the model chooses the ids,
 * so an unscoped lookup lets anyone read anything by pasting an id into Slack.
 */
export interface BotHost {
  botName: string;
  /** AI Gateway "provider/model" string, or any AI SDK LanguageModel. */
  model: string;
  systemPrompt: string;
  createChatState(): StateAdapter;
  installations: InstallationStore;
  approvals: ApprovalStore;
  approvalExecutors: Record<string, ApprovalExecutor>;
  /** The app's own session check, used by the OAuth routes. Null means signed out. */
  getCurrentUserId(): Promise<string | null>;
  findUserIdByEmail(email: string): Promise<string | null>;
  canAccess(userId: string, resourceId: string): Promise<boolean>;
  searchResources(userId: string, query: string): Promise<ResourceSummary[]>;
  getResource(userId: string, resourceId: string): Promise<ResourceDetail | null>;
  runAction(
    userId: string,
    resourceId: string,
    input: string | undefined,
    ctx: ActionContext,
  ): Promise<ActionOutcome>;
}

// ---------------------------------------------------------------------------
// Demo host. It makes the bot answer on day one; replace every body below.
// ---------------------------------------------------------------------------

const demoResources: (ResourceDetail & { ownerId: string })[] = [
  {
    id: "res_1",
    name: "Weekly competitor digest",
    kind: "report",
    content: "No competitor launched anything this week.",
    updatedAt: null,
    ownerId: "user_1",
  },
];

export const botHost: BotHost = {
  botName: "Assistant",
  model: "google/gemini-3.8-flash",
  systemPrompt: [
    "You are this product's Slack assistant. You answer questions about the user's resources and run actions on request.",
    "- Always look data up with the tools. Never guess or invent ids.",
    "- When the user names something, call search_resources to get its id first.",
    "- Call run_action AT MOST ONCE per request. On success reply with one line; on failure explain it and do not retry with different parameters.",
    "- After every tool call either answer in text or make exactly one more tool call.",
    "- Format for Slack: *bold*, bullets, short paragraphs. Summarize; never dump raw data.",
  ].join("\n"),
  // Memory state forgets thread subscriptions on every cold start. Production
  // needs a shared adapter; see the state section of setup.md.
  createChatState: () => createMemoryState(),
  installations: createMemoryInstallationStore(),
  approvals: createMemoryApprovalStore(),
  approvalExecutors: {
    demo_publish: async () => ({ ok: true }),
  },
  // Safe default: nobody can install until the real session check is wired in.
  async getCurrentUserId() {
    return null;
  },
  async findUserIdByEmail(email) {
    return email.endsWith("@example.com") ? "user_1" : null;
  },
  async canAccess(userId, resourceId) {
    return demoResources.some((r) => r.id === resourceId && r.ownerId === userId);
  },
  async searchResources(userId, query) {
    const q = query.toLowerCase();
    return demoResources
      .filter((r) => r.ownerId === userId && r.name.toLowerCase().includes(q))
      .map(({ id, name, kind }) => ({ id, name, kind }));
  },
  async getResource(userId, resourceId) {
    const found = demoResources.find((r) => r.id === resourceId && r.ownerId === userId);
    if (!found) return null;
    const { ownerId: _ownerId, ...detail } = found;
    return detail;
  },
  async runAction(userId, resourceId, input, ctx) {
    if (!(await this.canAccess(userId, resourceId))) {
      return { success: false, message: "You don't have access to that." };
    }
    if (!ctx.origin) {
      return { success: false, message: "This action needs a Slack thread to ask for approval in." };
    }
    await ctx.requestApproval(
      {
        kind: "demo_publish",
        resourceId,
        summary: "*Publish the weekly digest?*",
        payload: { note: input ?? null },
      },
      input ?? "No extra instructions.",
    );
    return { success: true, message: "I posted a preview above with Approve and Cancel buttons." };
  },
};
```

Notes on the parts that are easy to get wrong:

- **`getResource` returns `null` for both "missing" and "not yours".** Two
  different answers let the model, and so the user, probe which ids exist.
- **`runAction` re-checks access itself.** Do not rely on the model having
  called `search_resources` first; it may have been handed an id in the chat.
- **`model`** is a plain string resolved by the AI Gateway. Provider-specific
  options (the source app passed a Gemini thinking budget) belong next to
  `streamText` in `handlers.ts`, not in this interface.
- **`getCurrentUserId` defaults to `null`**, which makes the install route
  answer 401. That is the safe default; an accidental `return "user_1"` would
  attribute every installation to one account.

## Wiring

Singletons built from the host. The identity cache lives at module scope so it
survives across requests on a warm instance.

```ts
// lib/slack-bot/services.ts
import { createApprovalService } from "./approvals";
import { botHost } from "./host";
import { createSlackUserResolver } from "./identity";
import { postBlocks, updateBlocks } from "./outbound";

/** Module-level singletons: the identity cache must outlive a single request. */
export const resolveSlackUser = createSlackUserResolver({
  findUserIdByEmail: (email) => botHost.findUserIdByEmail(email),
});

export const approvalService = createApprovalService({
  store: botHost.approvals,
  installations: botHost.installations,
  poster: { postBlocks, updateBlocks },
  canAccess: (userId, resourceId) => botHost.canAccess(userId, resourceId),
  executors: botHost.approvalExecutors,
});
```

## Identity lookup

`findUserIdByEmail` must be **one indexed query**. The source app listed the
first 1000 auth users and filtered in memory, on every uncached message; user
1001 could never be matched and nothing said why.

Supabase has no admin "get user by email" call, so expose one through SQL.
*Not compiled or run as part of this skill's verification; review it against
your schema.*

```sql
create or replace function public.find_user_id_by_email(p_email text)
returns uuid
language sql
stable
security definer
set search_path = ''
as $$
  select id from auth.users where lower(email) = lower(p_email) limit 1
$$;

-- SECURITY DEFINER functions are executable by everyone unless revoked.
revoke execute on function public.find_user_id_by_email(text) from public, anon, authenticated;
grant execute on function public.find_user_id_by_email(text) to service_role;
```

```ts
async findUserIdByEmail(email) {
  const { data, error } = await adminClient.rpc("find_user_id_by_email", { p_email: email });
  if (error) throw new Error(`find_user_id_by_email failed: ${error.message}`);
  return (data as string | null) ?? null;
},
```

With an ORM it is `db.query.users.findFirst({ where: eq(sql`lower(email)`, email) })`
or the equivalent; make sure a functional index on `lower(email)` exists.

**The trust decision you are making:** a Slack profile email is accepted as
proof of identity. Ordinary members confirm their address, but organizations
that provision through SCIM or SSO set profile emails administratively. If any
workspace may install your app, treat the installing workspace's admins as able
to assert any email. When that is not acceptable, either restrict installation
to workspaces you approve, or replace email matching with explicit account
linking: the user confirms the link while signed in to the app, and you store
`(team_id, slack_user_id) → user_id`.

## Host probe

Run before generating files:

```bash
grep -E '"(next|chat|@chat-adapter/[a-z-]+|ai|zod|@supabase/supabase-js|drizzle-orm|@prisma/client)"' package.json
ls proxy.ts middleware.ts 2>/dev/null        # is /api/slack/* excluded from the auth proxy?
grep -rn "after(" app --include=route.ts | head -3
cat CLAUDE.md AGENTS.md 2>/dev/null | head -60
```

The second line matters more than it looks: an auth proxy that redirects
unauthenticated requests to a sign-in page will answer Slack's webhooks with a
302, and Slack reports only "your URL didn't respond". Exclude
`/api/slack/events` and `/api/slack/interactivity` from it. The signature is
their authentication.

New dependencies this module needs, to confirm with the user if absent:
`chat`, `@chat-adapter/slack`, one state adapter, `ai`, `zod`.

## Checklist

- [ ] Rename table confirmed with the user and applied to tool names and prompt
- [ ] Every `BotHost` body replaced; no demo data left
- [ ] `findUserIdByEmail` is a single indexed query
- [ ] `canAccess`, `getResource`, `searchResources`, `runAction` all scope by `userId`
- [ ] Webhook routes excluded from the host's auth proxy
- [ ] `BOT_STRINGS` in the product's language
- [ ] Production chat state adapter chosen — [setup.md](setup.md)
