# Approvals: buttons out, clicks back

The round trip. The model proposes, a human disposes: the app posts a preview
with **Approve** and **Cancel**, and the click returns as a webhook that runs the
side effect exactly once.

## State machine

```
            request()                 claim()                  finish()
 (nothing) ─────────▶ pending ─────────────────▶ processing ─────────▶ approved
                         │      one conditional      │                 cancelled
                         │      UPDATE wins          └───────────────▶ failed
                         └── every other click ──▶ "already decided"
```

`pending → processing` is the only transition that needs to be atomic, and it is
the whole design. Everything after it runs for exactly one caller.

## Service

```ts
// lib/slack-bot/approvals.ts
import type { SlackBlock } from "./outbound";
import { BOT_STRINGS } from "./strings";
import type {
  ApprovalRecord,
  ApprovalStore,
  InstallationStore,
  NewApproval,
} from "./types";

export const APPROVE_ACTION_ID = "approval.approve";
export const CANCEL_ACTION_ID = "approval.cancel";

/** Runs the approved side effect. Must not throw for expected failures; return them. */
export type ApprovalExecutor = (
  approval: ApprovalRecord,
) => Promise<{ ok: true } | { ok: false; error: string }>;

export type DecideResult =
  | { ok: true; approval: ApprovalRecord }
  | {
      ok: false;
      reason: "not_found" | "forbidden" | "already_decided" | "execution_failed";
      message: string;
    };

/** Injected so the service is testable without Slack. `outbound.ts` satisfies it. */
export interface ApprovalPoster {
  postBlocks(
    token: string,
    channelId: string,
    blocks: SlackBlock[],
    fallbackText: string,
    options?: { threadTs?: string | null },
  ): Promise<string>;
  updateBlocks(
    token: string,
    channelId: string,
    ts: string,
    blocks: SlackBlock[],
    fallbackText: string,
  ): Promise<void>;
}

interface ApprovalServiceDeps {
  store: ApprovalStore;
  installations: InstallationStore;
  poster: ApprovalPoster;
  canAccess: (userId: string, resourceId: string) => Promise<boolean>;
  executors: Record<string, ApprovalExecutor>;
}

const PREVIEW_MAX = 1500;

export function buildApprovalBlocks(approval: ApprovalRecord, preview: string): SlackBlock[] {
  const shown =
    preview.length > PREVIEW_MAX
      ? `${preview.slice(0, PREVIEW_MAX)}\n${BOT_STRINGS.previewTruncated}`
      : preview;
  return [
    { type: "section", text: { type: "mrkdwn", text: approval.summary } },
    { type: "divider" },
    { type: "section", text: { type: "mrkdwn", text: "```" + shown + "```" } },
    {
      type: "actions",
      block_id: `approval:${approval.id}`,
      elements: [
        {
          type: "button",
          style: "primary",
          text: { type: "plain_text", text: BOT_STRINGS.approveButton, emoji: true },
          action_id: APPROVE_ACTION_ID,
          // The value is an opaque id. Never put the payload here: anyone in the
          // channel can read it, and a client can send any value back.
          value: approval.id,
        },
        {
          type: "button",
          style: "danger",
          text: { type: "plain_text", text: BOT_STRINGS.cancelButton, emoji: true },
          action_id: CANCEL_ACTION_ID,
          value: approval.id,
        },
      ],
    },
  ];
}

/** The decided message has no actions block, so the buttons cannot be clicked again. */
export function buildDecidedBlocks(approval: ApprovalRecord): SlackBlock[] {
  const headline =
    approval.status === "approved"
      ? BOT_STRINGS.approvalApproved
      : approval.status === "cancelled"
        ? BOT_STRINGS.approvalCancelled
        : `${BOT_STRINGS.approvalFailed} ${approval.error ?? ""}`.trim();
  return [
    { type: "section", text: { type: "mrkdwn", text: headline } },
    { type: "context", elements: [{ type: "mrkdwn", text: approval.summary }] },
  ];
}

export function createApprovalService(deps: ApprovalServiceDeps) {
  const { store, installations, poster, canAccess, executors } = deps;

  async function refreshMessage(approval: ApprovalRecord): Promise<void> {
    if (!approval.messageTs) return;
    try {
      const installation = await installations.get(approval.origin.teamId);
      if (!installation) return;
      await poster.updateBlocks(
        installation.botToken,
        approval.origin.channelId,
        approval.messageTs,
        buildDecidedBlocks(approval),
        approval.summary,
      );
    } catch (err) {
      // The decision is already durable. A stale message is cosmetic: a second
      // click lands on `already_decided`.
      console.error("[slack-bot] failed to refresh approval message", approval.id, err);
    }
  }

  return {
    /** Stores the request, then posts the preview with Approve / Cancel into the origin thread. */
    async request(input: NewApproval, preview: string): Promise<ApprovalRecord> {
      if (!executors[input.kind]) {
        throw new Error(`No approval executor registered for kind "${input.kind}"`);
      }
      const installation = await installations.get(input.origin.teamId);
      if (!installation) {
        throw new Error(`No Slack installation for team ${input.origin.teamId}`);
      }
      // Row first, message second: a button must never exist without its row.
      const approval = await store.create(input);
      const ts = await poster.postBlocks(
        installation.botToken,
        input.origin.channelId,
        buildApprovalBlocks(approval, preview),
        approval.summary,
        { threadTs: input.origin.threadTs },
      );
      await store.attachMessage(approval.id, ts);
      return { ...approval, messageTs: ts };
    },

    /** Handles one button click. Safe under double clicks and concurrent approvers. */
    async decide(input: {
      approvalId: string;
      action: "approve" | "cancel";
      decidedByUserId: string;
    }): Promise<DecideResult> {
      const existing = await store.get(input.approvalId);
      if (!existing) {
        return { ok: false, reason: "not_found", message: BOT_STRINGS.approvalNotFound };
      }
      // Being in the channel is not authorization. Check the clicker, not the requester.
      if (!(await canAccess(input.decidedByUserId, existing.resourceId))) {
        return { ok: false, reason: "forbidden", message: BOT_STRINGS.approvalForbidden };
      }

      const claimed = await store.claim(input.approvalId, input.decidedByUserId);
      if (!claimed) {
        return {
          ok: false,
          reason: "already_decided",
          message: BOT_STRINGS.approvalAlreadyDecided,
        };
      }

      if (input.action === "cancel") {
        const cancelled = (await store.finish(claimed.id, "cancelled")) ?? claimed;
        await refreshMessage(cancelled);
        return { ok: true, approval: cancelled };
      }

      let outcome: { ok: true } | { ok: false; error: string };
      try {
        const executor = executors[claimed.kind];
        outcome = executor
          ? await executor(claimed)
          : { ok: false, error: `No executor for kind "${claimed.kind}"` };
      } catch (err) {
        outcome = { ok: false, error: err instanceof Error ? err.message : String(err) };
      }

      const finished =
        (await store.finish(
          claimed.id,
          outcome.ok ? "approved" : "failed",
          outcome.ok ? undefined : outcome.error,
        )) ?? claimed;
      await refreshMessage(finished);

      return outcome.ok
        ? { ok: true, approval: finished }
        : { ok: false, reason: "execution_failed", message: outcome.error };
    },
  };
}

export type ApprovalService = ReturnType<typeof createApprovalService>;
```

Why it is shaped this way:

- **Claim, then act.** The source app loaded the draft, checked
  `status === "draft"` in application code, sent the email, then wrote
  `status: "sent"`. Two clicks a moment apart both pass the check and both send.
  The slow path made a second click likely: the send ran *before* Slack was
  acknowledged, Slack showed "operation timed out" after three seconds, and the
  natural response to that is to click again. The customer gets the offer twice.
- **Row first, message second.** A posted button whose row failed to insert is a
  button that can only ever answer "not found".
- **Authorize the clicker.** Anyone in the channel can click. The check is
  against `decidedByUserId`, not the requester.
- **Access check before the claim.** An outsider's click must not consume the
  claim and lock out the people who can approve.
- **The decided message has no `actions` block.** Removing the buttons is the
  user-visible half of idempotency; the claim is the half that can be trusted.
- **Executor failures are terminal, not retried.** `failed` is shown in the
  thread with the error. Whether a send half-happened is unknowable from here;
  a human asks the bot again, which creates a new approval with a new preview.
- **`refreshMessage` swallows its error.** The decision is already durable. A
  message that still shows buttons costs one "already decided" reply.

## Requesting an approval

From a host action — this is the demo `runAction` in
[adaptation.md](adaptation.md) with a real payload:

```ts
await ctx.requestApproval(
  {
    kind: "send_email",
    resourceId,
    summary: `*Offer draft ready*\n*To:* ${draft.to}\n*Subject:* ${draft.subject}`,
    payload: draft, // frozen now: what is previewed is what is sent
  },
  draft.text,
);
return { success: true, message: "I posted the draft above with Approve and Cancel buttons." };
```

and its executor in `host.ts`:

```ts
approvalExecutors: {
  send_email: async (approval) => {
    const draft = approval.payload as { to: string; subject: string; html: string };
    const sent = await sendEmail(draft);
    return sent.ok ? { ok: true } : { ok: false, error: sent.error };
  },
},
```

The tool's return message matters: the model relays it, so the user is told to
look at the buttons rather than being told the email was sent.

## Signature verification

The events route is verified by the Chat SDK. The interactivity route is yours,
so it verifies by hand.

```ts
// lib/slack-bot/verify-signature.ts
import { createHmac, timingSafeEqual } from "node:crypto";

const MAX_SKEW_SECONDS = 5 * 60;

/**
 * Verifies Slack's `v0` request signature over the RAW body. Read the body with
 * `request.text()` before anything parses it: a re-serialized body never matches.
 */
export function verifySlackSignature(input: {
  signingSecret: string;
  body: string;
  timestamp: string | null;
  signature: string | null;
  nowMs?: number;
}): boolean {
  const { signingSecret, body, timestamp, signature } = input;
  if (!signingSecret) throw new Error("Slack signing secret is not configured");
  if (!timestamp || !signature) return false;

  // parseInt would accept "1700000000abc"; the header must be digits only.
  if (!/^\d+$/.test(timestamp)) return false;
  const nowSeconds = Math.floor((input.nowMs ?? Date.now()) / 1000);
  if (Math.abs(nowSeconds - Number(timestamp)) > MAX_SKEW_SECONDS) return false;

  const hmac = createHmac("sha256", signingSecret)
    .update(`v0:${timestamp}:${body}`)
    .digest("hex");
  const expected = Buffer.from(`v0=${hmac}`, "utf8");
  const given = Buffer.from(signature, "utf8");

  // Compare BYTE lengths. Equal string lengths with a multi-byte character make
  // timingSafeEqual throw, which turns a forged request into a 500 instead of a 401.
  if (expected.length !== given.length) return false;
  return timingSafeEqual(expected, given);
}
```

Two fixes over the source, both tested:

- It compared **string** lengths, then called `timingSafeEqual` on buffers. A
  forged signature of the right character count containing one multi-byte
  character has a different byte length; `timingSafeEqual` throws, and the
  route answers 500 instead of 401. Harmless to data, but it is an
  unauthenticated way to make the endpoint throw.
- `parseInt("1700000000abc")` is a valid timestamp to JavaScript. The header is
  now digits or nothing.

The secret is a parameter so the function is pure and testable.

## Interactivity route

```ts
// app/api/slack/interactivity/route.ts
import { after } from "next/server";

import { APPROVE_ACTION_ID, CANCEL_ACTION_ID } from "@/lib/slack-bot/approvals";
import { botHost } from "@/lib/slack-bot/host";
import { respondEphemeral } from "@/lib/slack-bot/outbound";
import { approvalService, resolveSlackUser } from "@/lib/slack-bot/services";
import { BOT_STRINGS } from "@/lib/slack-bot/strings";
import { verifySlackSignature } from "@/lib/slack-bot/verify-signature";

export const maxDuration = 300;

interface BlockActionsPayload {
  type: string;
  user?: { id?: string; team_id?: string };
  team?: { id?: string };
  response_url?: string;
  actions?: { action_id?: string; value?: string }[];
}

export async function POST(request: Request): Promise<Response> {
  // Raw text first. Parsing the form before verifying breaks the signature.
  const rawBody = await request.text();
  const verified = verifySlackSignature({
    signingSecret: process.env.SLACK_SIGNING_SECRET ?? "",
    body: rawBody,
    timestamp: request.headers.get("x-slack-request-timestamp"),
    signature: request.headers.get("x-slack-signature"),
  });
  if (!verified) return Response.json({ error: "invalid signature" }, { status: 401 });

  // Interactions arrive form-encoded with the JSON inside a `payload` field.
  let payload: BlockActionsPayload;
  try {
    payload = JSON.parse(new URLSearchParams(rawBody).get("payload") ?? "") as BlockActionsPayload;
  } catch {
    return Response.json({ error: "invalid payload" }, { status: 400 });
  }

  // Acknowledge inside Slack's 3-second window, then work. A send that runs
  // before the ack shows the user a timeout warning, and they click again.
  if (payload.type === "block_actions") after(() => handleBlockActions(payload));
  return new Response("", { status: 200 });
}

async function handleBlockActions(payload: BlockActionsPayload): Promise<void> {
  const reply = async (text: string): Promise<void> => {
    if (payload.response_url) await respondEphemeral(payload.response_url, text).catch(() => {});
  };

  try {
    const teamId = payload.team?.id ?? payload.user?.team_id ?? "";
    const slackUserId = payload.user?.id ?? "";
    const installation = await botHost.installations.get(teamId);
    if (!installation || !slackUserId) return;

    const identity = await resolveSlackUser(installation.botToken, teamId, slackUserId);
    if (!identity.ok) {
      await reply(
        identity.reason === "no_email" ? BOT_STRINGS.identityNoEmail : BOT_STRINGS.identityNoMatch,
      );
      return;
    }

    for (const action of payload.actions ?? []) {
      const isApprove = action.action_id === APPROVE_ACTION_ID;
      if (!isApprove && action.action_id !== CANCEL_ACTION_ID) continue;
      if (!action.value) continue;

      const result = await approvalService.decide({
        approvalId: action.value,
        action: isApprove ? "approve" : "cancel",
        decidedByUserId: identity.userId,
      });
      // The click must never vanish: tell the clicker, privately, why nothing happened.
      if (!result.ok) await reply(result.message);
    }
  } catch (err) {
    console.error("[slack-bot] interactivity failed", err);
    await reply(BOT_STRINGS.answerFailed);
  }
}
```

- **Ack first.** `after()` defers the work until the 200 has gone out. The
  source did identity resolution, the email send and two database writes before
  responding.
- **`response_url` for the unhappy paths.** The source logged a refused click
  and returned 200; the person clicked a button and nothing happened, with no
  way to learn why. An ephemeral reply is visible only to the clicker, needs no
  token, and works in channels the bot cannot otherwise post to. *This is an
  addition; the source had no user feedback on refusal.*
- **Unknown `action_id`s are skipped, not errors.** Other features will add
  buttons, and this route receives all of them.
- **Non-`block_actions` payloads** (`view_submission`, shortcuts) are
  acknowledged and ignored.

### The SDK alternative

The Chat SDK can receive interactions itself: point Slack's interactivity URL at
the events route and register `bot.onAction([APPROVE_ACTION_ID, CANCEL_ACTION_ID], handler)`.
That removes this route and the hand-rolled verification. *It was not used in
the source app and has not been exercised by this skill's verification.* The
service above is independent of the transport; only the route changes.

## Checklist

- [ ] Button `value` is the approval id and nothing else
- [ ] `claim` is one conditional write — [data-model.md](data-model.md)
- [ ] Clicker resolved to an app user and checked against the resource
- [ ] 200 returned before any slow work
- [ ] Refusals answered through `response_url`
- [ ] Decided message rewritten without buttons
- [ ] Signature verified over the raw body, before parsing
- [ ] Route excluded from the app's auth proxy
