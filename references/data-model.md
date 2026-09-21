# Data model

Two tables. Chat state (subscriptions, locks, dedupe) is **not** here; it lives
in the Chat SDK state adapter — [setup.md](setup.md).

## Shapes

```ts
// lib/slack-bot/types.ts
/** One row per Slack workspace. The bot token belongs to the workspace, not to whoever clicked Install. */
export interface SlackInstallationRecord {
  teamId: string;
  teamName: string | null;
  botToken: string;
  botUserId: string | null;
  /** App user who ran the OAuth flow. Audit trail only, never used to pick a token. */
  installedByUserId: string | null;
}

export interface InstallationStore {
  get(teamId: string): Promise<SlackInstallationRecord | null>;
  upsert(record: SlackInstallationRecord): Promise<void>;
  remove(teamId: string): Promise<void>;
}

/** Where in Slack a conversation is happening. Enough to post back into it later. */
export interface SlackOrigin {
  teamId: string;
  channelId: string;
  /** Thread root. Null means "post at channel level". */
  threadTs: string | null;
}

export type ApprovalStatus =
  | "pending"
  | "processing"
  | "approved"
  | "cancelled"
  | "failed";

export interface ApprovalRecord {
  id: string;
  /** Selects the executor that runs on approve, e.g. "send_email". */
  kind: string;
  /** What `canAccess` is checked against when someone clicks a button. */
  resourceId: string;
  /** One-line mrkdwn headline shown above the preview and kept after the decision. */
  summary: string;
  /** Everything the executor needs. Frozen at request time: what was previewed is what runs. */
  payload: unknown;
  status: ApprovalStatus;
  origin: SlackOrigin;
  /** ts of the message carrying the buttons, so it can be rewritten after the decision. */
  messageTs: string | null;
  requestedByUserId: string;
  decidedByUserId: string | null;
  error: string | null;
}

export type NewApproval = Pick<
  ApprovalRecord,
  "kind" | "resourceId" | "summary" | "payload" | "origin" | "requestedByUserId"
>;

export interface ApprovalStore {
  create(input: NewApproval): Promise<ApprovalRecord>;
  get(id: string): Promise<ApprovalRecord | null>;
  attachMessage(id: string, messageTs: string): Promise<void>;
  /**
   * Atomically moves pending -> processing. Returns the claimed record, or null
   * when someone else already claimed it. Must be ONE conditional write.
   */
  claim(id: string, decidedByUserId: string): Promise<ApprovalRecord | null>;
  finish(
    id: string,
    status: "approved" | "cancelled" | "failed",
    error?: string,
  ): Promise<ApprovalRecord | null>;
}
```

## Why these shapes

| Decision | Reason |
|---|---|
| `slack_installations` keyed by `team_id` alone | A bot token belongs to the workspace. The source app stored tokens per `(user, provider, team)` and picked one with an unordered `limit(1)`; when two people connected the same workspace, which row won varied between queries. |
| `installed_by_user_id` is audit only | Picking a token through "the owner's integration" breaks the day the owner leaves. |
| `status` has a `processing` state | It is the claim. `pending → processing` is the one transition that must be atomic; see [approvals.md](approvals.md). |
| `payload` frozen at request time | What the human previewed is what runs. Re-deriving at approve time lets the data change between preview and send. |
| `origin` stored on the approval | The click arrives with no memory of the conversation; the row is how the app finds the message to rewrite. |
| Approvals are their own table | The source app kept draft state inside the JSON of a job-result row and updated it by read-modify-write. That cannot be claimed atomically. |

## Schema (Postgres)

*SQL is not executed by this skill's verification. Review before applying.*

```sql
create table public.slack_installations (
  team_id              text primary key,
  team_name            text,
  bot_token            text not null,
  bot_user_id          text,
  installed_by_user_id uuid references auth.users (id) on delete set null,
  created_at           timestamptz not null default now(),
  updated_at           timestamptz not null default now()
);

create table public.slack_approvals (
  id                   uuid primary key default gen_random_uuid(),
  kind                 text not null,
  resource_id          text not null,
  summary              text not null,
  payload              jsonb,
  status               text not null default 'pending'
                       check (status in ('pending','processing','approved','cancelled','failed')),
  team_id              text not null,
  channel_id           text not null,
  thread_ts            text,
  message_ts           text,
  requested_by_user_id uuid not null,
  decided_by_user_id   uuid,
  decided_at           timestamptz,
  error                text,
  created_at           timestamptz not null default now()
);

-- The operator's "what is stuck" query; see operations.md.
create index slack_approvals_open_idx
  on public.slack_approvals (status, created_at)
  where status in ('pending', 'processing');

-- RLS on, no policies: only the service role reads or writes. Slack webhooks
-- carry no user session, so there is nobody for a policy to authorize.
alter table public.slack_installations enable row level security;
alter table public.slack_approvals    enable row level security;
```

`bot_token` is a live credential. At minimum keep the table service-role only,
as above. Encrypting the column (pgsodium / Vault, or app-level AES-GCM with a
key in env) is recommended and was **not** done in the source app.

Without Supabase, drop the `references auth.users` clause and point
`installed_by_user_id` at your own users table.

## Supabase implementation

```ts
// lib/slack-bot/stores-supabase.ts
import type { SupabaseClient } from "@supabase/supabase-js";

import type {
  ApprovalRecord,
  ApprovalStore,
  InstallationStore,
  SlackInstallationRecord,
} from "./types";

// Pass a SERVICE-ROLE client. Slack webhooks carry no user session, so RLS has
// nobody to authorize; both tables stay closed to the anon and authenticated roles.

interface InstallationRow {
  team_id: string;
  team_name: string | null;
  bot_token: string;
  bot_user_id: string | null;
  installed_by_user_id: string | null;
}

export function createSupabaseInstallationStore(db: SupabaseClient): InstallationStore {
  const toRecord = (row: InstallationRow): SlackInstallationRecord => ({
    teamId: row.team_id,
    teamName: row.team_name,
    botToken: row.bot_token,
    botUserId: row.bot_user_id,
    installedByUserId: row.installed_by_user_id,
  });
  return {
    async get(teamId) {
      const { data, error } = await db
        .from("slack_installations")
        .select("team_id, team_name, bot_token, bot_user_id, installed_by_user_id")
        .eq("team_id", teamId)
        .maybeSingle<InstallationRow>();
      if (error) throw new Error(`slack_installations read failed: ${error.message}`);
      return data ? toRecord(data) : null;
    },
    async upsert(record) {
      const { error } = await db.from("slack_installations").upsert(
        {
          team_id: record.teamId,
          team_name: record.teamName,
          bot_token: record.botToken,
          bot_user_id: record.botUserId,
          installed_by_user_id: record.installedByUserId,
          updated_at: new Date().toISOString(),
        },
        { onConflict: "team_id" },
      );
      if (error) throw new Error(`slack_installations write failed: ${error.message}`);
    },
    async remove(teamId) {
      const { error } = await db.from("slack_installations").delete().eq("team_id", teamId);
      if (error) throw new Error(`slack_installations delete failed: ${error.message}`);
    },
  };
}

interface ApprovalRow {
  id: string;
  kind: string;
  resource_id: string;
  summary: string;
  payload: unknown;
  status: ApprovalRecord["status"];
  team_id: string;
  channel_id: string;
  thread_ts: string | null;
  message_ts: string | null;
  requested_by_user_id: string;
  decided_by_user_id: string | null;
  error: string | null;
}

const APPROVAL_COLUMNS =
  "id, kind, resource_id, summary, payload, status, team_id, channel_id, thread_ts, message_ts, requested_by_user_id, decided_by_user_id, error";

export function createSupabaseApprovalStore(db: SupabaseClient): ApprovalStore {
  const toRecord = (row: ApprovalRow): ApprovalRecord => ({
    id: row.id,
    kind: row.kind,
    resourceId: row.resource_id,
    summary: row.summary,
    payload: row.payload,
    status: row.status,
    origin: { teamId: row.team_id, channelId: row.channel_id, threadTs: row.thread_ts },
    messageTs: row.message_ts,
    requestedByUserId: row.requested_by_user_id,
    decidedByUserId: row.decided_by_user_id,
    error: row.error,
  });
  return {
    async create(input) {
      const { data, error } = await db
        .from("slack_approvals")
        .insert({
          kind: input.kind,
          resource_id: input.resourceId,
          summary: input.summary,
          payload: input.payload,
          team_id: input.origin.teamId,
          channel_id: input.origin.channelId,
          thread_ts: input.origin.threadTs,
          requested_by_user_id: input.requestedByUserId,
        })
        .select(APPROVAL_COLUMNS)
        .single<ApprovalRow>();
      if (error || !data) throw new Error(`slack_approvals insert failed: ${error?.message}`);
      return toRecord(data);
    },
    async get(id) {
      const { data, error } = await db
        .from("slack_approvals")
        .select(APPROVAL_COLUMNS)
        .eq("id", id)
        .maybeSingle<ApprovalRow>();
      if (error) throw new Error(`slack_approvals read failed: ${error.message}`);
      return data ? toRecord(data) : null;
    },
    async attachMessage(id, messageTs) {
      const { error } = await db
        .from("slack_approvals")
        .update({ message_ts: messageTs })
        .eq("id", id);
      if (error) throw new Error(`slack_approvals update failed: ${error.message}`);
    },
    async claim(id, decidedByUserId) {
      // The status filter is the lock: Postgres re-checks it under the row lock,
      // so of two concurrent clicks exactly one UPDATE matches a row.
      const { data, error } = await db
        .from("slack_approvals")
        .update({
          status: "processing",
          decided_by_user_id: decidedByUserId,
          decided_at: new Date().toISOString(),
        })
        .eq("id", id)
        .eq("status", "pending")
        .select(APPROVAL_COLUMNS)
        .maybeSingle<ApprovalRow>();
      if (error) throw new Error(`slack_approvals claim failed: ${error.message}`);
      return data ? toRecord(data) : null;
    },
    async finish(id, status, error) {
      const { data, error: dbError } = await db
        .from("slack_approvals")
        .update({ status, error: error ?? null })
        .eq("id", id)
        .eq("status", "processing")
        .select(APPROVAL_COLUMNS)
        .maybeSingle<ApprovalRow>();
      if (dbError) throw new Error(`slack_approvals finish failed: ${dbError.message}`);
      return data ? toRecord(data) : null;
    },
  };
}
```

## In-memory implementation

For tests and a first local run. It is also the executable specification of the
`claim` contract.

```ts
// lib/slack-bot/stores-memory.ts
import type {
  ApprovalRecord,
  ApprovalStore,
  InstallationStore,
  SlackInstallationRecord,
} from "./types";

/** Tests and local demos only: a serverless instance forgets everything. */
export function createMemoryInstallationStore(): InstallationStore {
  const rows = new Map<string, SlackInstallationRecord>();
  return {
    async get(teamId) {
      return rows.get(teamId) ?? null;
    },
    async upsert(record) {
      rows.set(record.teamId, { ...record });
    },
    async remove(teamId) {
      rows.delete(teamId);
    },
  };
}

export function createMemoryApprovalStore(): ApprovalStore {
  const rows = new Map<string, ApprovalRecord>();
  let seq = 0;
  return {
    async create(input) {
      const record: ApprovalRecord = {
        ...input,
        id: `apr_${++seq}`,
        status: "pending",
        messageTs: null,
        decidedByUserId: null,
        error: null,
      };
      rows.set(record.id, record);
      return { ...record };
    },
    async get(id) {
      const row = rows.get(id);
      return row ? { ...row } : null;
    },
    async attachMessage(id, messageTs) {
      const row = rows.get(id);
      if (row) row.messageTs = messageTs;
    },
    async claim(id, decidedByUserId) {
      const row = rows.get(id);
      // Check and write with no await between them: atomic on one event loop.
      if (!row || row.status !== "pending") return null;
      row.status = "processing";
      row.decidedByUserId = decidedByUserId;
      return { ...row };
    },
    async finish(id, status, error) {
      const row = rows.get(id);
      if (!row || row.status !== "processing") return null;
      row.status = status;
      row.error = error ?? null;
      return { ...row };
    },
  };
}
```

## Other data layers

The only operation that needs care is `claim`. Everything else is plain CRUD.
*These snippets are not compiled by this skill's verification.*

| Layer | `claim` |
|---|---|
| Drizzle | `db.update(approvals).set({ status: "processing", decidedByUserId }).where(and(eq(approvals.id, id), eq(approvals.status, "pending"))).returning()` — empty array means lost the race |
| Prisma | `updateMany({ where: { id, status: "pending" }, data: {…} })`, then check `count === 1` and re-read. `update` throws on no match; `updateMany` does not |
| Raw SQL | `update slack_approvals set status='processing', … where id=$1 and status='pending' returning *` |
| Firestore | A transaction that reads the doc and throws unless `status === "pending"`. A plain `update` has no precondition on field values |

What never works: `select`, check `status` in application code, then `update`.
Two requests both pass the check.

## Checklist

- [ ] `slack_installations` has one row per workspace, keyed by `team_id`
- [ ] Both tables closed to anon and authenticated roles
- [ ] `claim` is a single conditional write in your data layer
- [ ] `finish` only moves rows out of `processing`
- [ ] Decision made, and written down, about encrypting `bot_token`
