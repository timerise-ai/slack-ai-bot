# Testing

31 tests over the logic this skill claims is correct. They ran green, and the
templates compiled under `strict` and `--noUncheckedIndexedAccess`, against
`chat` 4.40, `ai` 7, `zod` 4, `next` 16 and `@supabase/supabase-js` 2.

**Not covered**, and stated rather than implied: the SQL in
[data-model.md](data-model.md) and [adaptation.md](adaptation.md) was not
executed; the Redis and Postgres state adapter snippets and the Drizzle / Prisma
`claim` snippets were not compiled; nothing here talks to a real Slack
workspace. `handlers.ts`, `bot.ts` and the routes are type-checked only. The
end-to-end pass is the manual script at the bottom.

## Running

Written for `bun:test`. For vitest, change the one import to
`import { describe, expect, it } from "vitest";`; the API used is common to both.
That line is the only one that ever changes: the suite is not moved, split,
converted to another runner or given extra cases, and it reports 31. Tests of
your own go in a file of their own beside it. With neither runner installed,
install one as a dev dependency (`npm i -D vitest`); the package registry is
not an external service.

```bash
bun test lib/slack-bot
```

## What each group proves

| Group | Claim under test |
|---|---|
| `verifySlackSignature` | Tampering, replay, junk timestamps rejected; a multi-byte forgery returns `false` instead of throwing |
| `extractSlackOrigin` | Reply vs thread root; never an origin with empty ids |
| `createSlackUserResolver` | Case-insensitive; cache is per workspace, expires, and never stores a miss; missing scope is its own reason |
| OAuth helpers | Required scopes present; open-redirect inputs collapse to `/`; query strings merge |
| `markdownToMrkdwn` | The three converter regressions, plus code and tables left intact |
| outbound | No chunk over the limit and no text lost; 50-block ceiling with a visible note; 429 retried once; `ok:false` throws |
| approval round trip | Opaque button value; **one execution under three concurrent clicks**; outsider refused without consuming the claim; cancel is final; thrown executor recorded and buttons removed |

The concurrent-click test is the one to keep if you keep only one. Port it to
your real `ApprovalStore` as an integration test: it is the only thing that
proves your data layer's `claim` is atomic.

## The suite

```ts
// lib/slack-bot/slack-bot.test.ts
import { createHmac } from "node:crypto";

import { describe, expect, it } from "bun:test";

import { createApprovalService, type ApprovalPoster } from "./approvals";
import { extractSlackOrigin } from "./context";
import { createSlackUserResolver } from "./identity";
import { markdownToMrkdwn } from "./mrkdwn";
import { safeReturnPath, SLACK_BOT_SCOPES, withParam } from "./oauth";
import { buildTextBlocks, slackApi, splitIntoChunks } from "./outbound";
import {
  createMemoryApprovalStore,
  createMemoryInstallationStore,
} from "./stores-memory";
import { verifySlackSignature } from "./verify-signature";

const SECRET = "shhh";
const NOW_MS = 1_700_000_000_000;
const TS = String(NOW_MS / 1000);
const sign = (body: string, ts = TS): string =>
  `v0=${createHmac("sha256", SECRET).update(`v0:${ts}:${body}`).digest("hex")}`;

describe("verifySlackSignature", () => {
  const base = { signingSecret: SECRET, body: "payload=1", timestamp: TS, nowMs: NOW_MS };

  it("accepts a correctly signed request", () => {
    expect(verifySlackSignature({ ...base, signature: sign("payload=1") })).toBe(true);
  });
  it("rejects a tampered body", () => {
    expect(verifySlackSignature({ ...base, body: "payload=2", signature: sign("payload=1") })).toBe(false);
  });
  it("rejects a replay older than five minutes", () => {
    const old = String(NOW_MS / 1000 - 301);
    expect(verifySlackSignature({ ...base, timestamp: old, signature: sign("payload=1", old) })).toBe(false);
  });
  it("rejects a timestamp with trailing junk instead of parsing its prefix", () => {
    expect(verifySlackSignature({ ...base, timestamp: `${TS}abc`, signature: sign("payload=1") })).toBe(false);
  });
  it("returns false, not a thrown RangeError, for a multi-byte signature of equal string length", () => {
    const forged = `v0=${"a".repeat(63)}é`;
    expect(forged.length).toBe(sign("payload=1").length);
    expect(verifySlackSignature({ ...base, signature: forged })).toBe(false);
  });
  it("rejects missing headers", () => {
    expect(verifySlackSignature({ ...base, signature: null })).toBe(false);
    expect(verifySlackSignature({ ...base, timestamp: null, signature: sign("payload=1") })).toBe(false);
  });
});

describe("extractSlackOrigin", () => {
  it("uses thread_ts for a reply and the message's own ts for a thread root", () => {
    expect(extractSlackOrigin({ team: "T1", channel: "C1", thread_ts: "1.0", ts: "2.0" })).toEqual({
      teamId: "T1", channelId: "C1", threadTs: "1.0",
    });
    expect(extractSlackOrigin({ team_id: "T1", channel: "C1", ts: "2.0" })).toEqual({
      teamId: "T1", channelId: "C1", threadTs: "2.0",
    });
  });
  it("returns null rather than an origin with empty ids", () => {
    expect(extractSlackOrigin({ channel: "C1", ts: "2.0" })).toBeNull();
    expect(extractSlackOrigin({ team: "T1", ts: "2.0" })).toBeNull();
    expect(extractSlackOrigin(undefined)).toBeNull();
  });
});

function fakeUsersInfo(email: string | undefined, counter = { calls: 0 }) {
  const fetchImpl = (async () => {
    counter.calls++;
    return new Response(JSON.stringify({ ok: true, user: { profile: { email } } }));
  }) as unknown as typeof fetch;
  return { fetchImpl, counter };
}

describe("createSlackUserResolver", () => {
  it("matches case-insensitively and caches per workspace", async () => {
    const { fetchImpl, counter } = fakeUsersInfo("Ada@Example.com");
    const seen: string[] = [];
    const resolve = createSlackUserResolver({
      fetchImpl,
      findUserIdByEmail: async (email) => (seen.push(email), "user_1"),
    });
    expect(await resolve("xoxb", "T1", "U1")).toEqual({ ok: true, userId: "user_1" });
    expect(await resolve("xoxb", "T1", "U1")).toEqual({ ok: true, userId: "user_1" });
    expect(seen).toEqual(["ada@example.com"]);
    expect(counter.calls).toBe(1);
    await resolve("xoxb", "T2", "U1"); // same Slack id, other workspace: not a cache hit
    expect(counter.calls).toBe(2);
  });
  it("expires the cache so a removed user loses access", async () => {
    let clock = 0;
    let exists = true;
    const { fetchImpl } = fakeUsersInfo("ada@example.com");
    const resolve = createSlackUserResolver({
      fetchImpl, ttlMs: 1000, now: () => clock,
      findUserIdByEmail: async () => (exists ? "user_1" : null),
    });
    expect((await resolve("xoxb", "T1", "U1")).ok).toBe(true);
    exists = false;
    clock = 999;
    expect((await resolve("xoxb", "T1", "U1")).ok).toBe(true);
    clock = 1001;
    expect(await resolve("xoxb", "T1", "U1")).toEqual({ ok: false, reason: "no_matching_user" });
  });
  it("does not cache misses", async () => {
    let exists = false;
    const { fetchImpl } = fakeUsersInfo("ada@example.com");
    const resolve = createSlackUserResolver({ fetchImpl, findUserIdByEmail: async () => (exists ? "user_1" : null) });
    expect((await resolve("xoxb", "T1", "U1")).ok).toBe(false);
    exists = true;
    expect((await resolve("xoxb", "T1", "U1")).ok).toBe(true);
  });
  it("names the missing email scope as its own reason", async () => {
    const { fetchImpl } = fakeUsersInfo(undefined);
    const resolve = createSlackUserResolver({ fetchImpl, findUserIdByEmail: async () => "user_1" });
    expect(await resolve("xoxb", "T1", "U1")).toEqual({ ok: false, reason: "no_email" });
  });
});

describe("OAuth helpers", () => {
  it("requests the scopes identity and events depend on", () => {
    for (const scope of ["users:read.email", "app_mentions:read", "im:history", "chat:write"]) {
      expect(SLACK_BOT_SCOPES as readonly string[]).toContain(scope);
    }
  });
  it("only lets same-site paths through", () => {
    expect(safeReturnPath("/settings?tab=slack")).toBe("/settings?tab=slack");
    for (const bad of ["//evil.com", "/\\evil.com", "@evil.com", "https://evil.com", "", null,
      "/\t/evil.com", "/\n/evil.com", "/a/..//evil.com"]) {
      expect(safeReturnPath(bad)).toBe("/");
    }
  });
  it("appends to a path that already has a query and never leaves the app origin", () => {
    expect(withParam("https://app.test", "/s?tab=slack", "slack_connected", "1")).toBe(
      "https://app.test/s?tab=slack&slack_connected=1",
    );
    expect(new URL(withParam("https://app.test", "@evil.com", "x", "1")).host).toBe("app.test");
  });
});

describe("markdownToMrkdwn", () => {
  it("keeps bold bold", () => {
    expect(markdownToMrkdwn("**bold** and *italic*")).toBe("*bold* and _italic_");
  });
  it("converts bold-italic once", () => {
    expect(markdownToMrkdwn("***both***")).toBe("*_both_*");
  });
  it("does not double-wrap a heading that is already bold", () => {
    expect(markdownToMrkdwn("# **Title**")).toBe("*Title*");
    expect(markdownToMrkdwn("## Plain title")).toBe("*Plain title*");
  });
  it("converts links, bullets and strikethrough", () => {
    expect(markdownToMrkdwn("* item [docs](https://x.test/a)\n~~old~~")).toBe(
      "• item <https://x.test/a|docs>\n~old~",
    );
  });
  it("leaves code untouched and fences tables", () => {
    expect(markdownToMrkdwn("`**raw**`\n```\n**raw**\n```")).toBe("`**raw**`\n```\n**raw**\n```");
    expect(markdownToMrkdwn("| a | b |\n|---|---|\n| 1 | 2 |")).toBe("```\n| a | b |\n|---|---|\n| 1 | 2 |\n```");
  });
  it("does not read arithmetic as emphasis", () => {
    expect(markdownToMrkdwn("2 * 3 * 4")).toBe("2 * 3 * 4");
  });
});

describe("outbound", () => {
  it("never produces a chunk over the limit and loses no text", () => {
    const text = `${"a".repeat(2500)}\n\n${"b".repeat(2500)}\n${"c".repeat(7000)}`;
    const chunks = splitIntoChunks(text, 3000);
    expect(chunks.every((c) => c.length <= 3000)).toBe(true);
    expect(chunks.join("").replace(/\n/g, "")).toBe(text.replace(/\n/g, ""));
  });
  it("stays within 50 blocks and says so when it truncates", () => {
    const text = Array.from({ length: 80 }, () => "x".repeat(2999)).join("\n\n");
    const blocks = buildTextBlocks(text, { header: "Report", truncationNote: "cut" });
    expect(blocks.length).toBe(50);
    expect(JSON.stringify(blocks[49])).toContain("cut");
  });
  it("retries a 429 once after Retry-After, then succeeds", async () => {
    let calls = 0;
    const fetchImpl = (async () =>
      ++calls === 1
        ? new Response("", { status: 429, headers: { "retry-after": "0" } })
        : new Response(JSON.stringify({ ok: true, ts: "1.1" }))) as unknown as typeof fetch;
    expect((await slackApi("xoxb", "chat.postMessage", {}, fetchImpl)).ts).toBe("1.1");
    expect(calls).toBe(2);
  });
  it("throws on HTTP 200 with ok:false", async () => {
    const fetchImpl = (async () =>
      new Response(JSON.stringify({ ok: false, error: "channel_not_found" }))) as unknown as typeof fetch;
    await expect(slackApi("xoxb", "chat.postMessage", {}, fetchImpl)).rejects.toThrow("channel_not_found");
  });
});

async function approvalFixture(executor = async () => ({ ok: true as const })) {
  const installations = createMemoryInstallationStore();
  await installations.upsert({ teamId: "T1", teamName: "Acme", botToken: "xoxb", botUserId: "B1", installedByUserId: "user_1" });
  const posted: unknown[] = [];
  const updates: string[] = [];
  const poster: ApprovalPoster = {
    postBlocks: async (_t, _c, blocks) => (posted.push(blocks), "111.222"),
    updateBlocks: async (_t, _c, _ts, blocks) => void updates.push(JSON.stringify(blocks)),
  };
  let runs = 0;
  const service = createApprovalService({
    store: createMemoryApprovalStore(),
    installations,
    poster,
    canAccess: async (userId) => userId !== "outsider",
    executors: { send_email: async () => (runs++, executor()) },
  });
  const approval = await service.request(
    {
      kind: "send_email", resourceId: "res_1", summary: "*Send offer?*", payload: { to: "a@b.c" },
      origin: { teamId: "T1", channelId: "C1", threadTs: "1.0" }, requestedByUserId: "user_1",
    },
    "Hello",
  );
  return { service, approval, posted, updates, runs: () => runs };
}

describe("approval round trip", () => {
  it("posts buttons carrying only the opaque id", async () => {
    const { approval, posted } = await approvalFixture();
    expect(approval.messageTs).toBe("111.222");
    const json = JSON.stringify(posted[0]);
    expect(json).toContain(`"value":"${approval.id}"`);
    expect(json).not.toContain("a@b.c");
  });
  it("runs the side effect exactly once under a double click", async () => {
    const f = await approvalFixture();
    const click = () => f.service.decide({ approvalId: f.approval.id, action: "approve", decidedByUserId: "user_1" });
    const results = await Promise.all([click(), click(), click()]);
    expect(f.runs()).toBe(1);
    expect(results.filter((r) => r.ok).length).toBe(1);
    expect(results.filter((r) => !r.ok && r.reason === "already_decided").length).toBe(2);
  });
  it("refuses a clicker without access and leaves the request pending", async () => {
    const f = await approvalFixture();
    const denied = await f.service.decide({ approvalId: f.approval.id, action: "approve", decidedByUserId: "outsider" });
    expect(denied.ok === false && denied.reason).toBe("forbidden");
    expect(f.runs()).toBe(0);
    expect((await f.service.decide({ approvalId: f.approval.id, action: "approve", decidedByUserId: "user_1" })).ok).toBe(true);
  });
  it("cancel never runs the executor, and approve after cancel is refused", async () => {
    const f = await approvalFixture();
    expect((await f.service.decide({ approvalId: f.approval.id, action: "cancel", decidedByUserId: "user_1" })).ok).toBe(true);
    const late = await f.service.decide({ approvalId: f.approval.id, action: "approve", decidedByUserId: "user_1" });
    expect(late.ok === false && late.reason).toBe("already_decided");
    expect(f.runs()).toBe(0);
  });
  it("records a thrown executor as failed and rewrites the message without buttons", async () => {
    const f = await approvalFixture(async () => { throw new Error("smtp down"); });
    const result = await f.service.decide({ approvalId: f.approval.id, action: "approve", decidedByUserId: "user_1" });
    expect(result.ok === false && result.reason).toBe("execution_failed");
    expect(f.updates[0]).toContain("smtp down");
    expect(f.updates[0]).not.toContain('"type":"actions"');
  });
  it("rejects a request for a kind with no executor before posting anything", async () => {
    const f = await approvalFixture();
    await expect(
      f.service.request({ kind: "nope", resourceId: "r", summary: "s", payload: null, origin: { teamId: "T1", channelId: "C1", threadTs: null }, requestedByUserId: "user_1" }, "p"),
    ).rejects.toThrow("No approval executor");
    expect(f.posted.length).toBe(1);
  });
});
```

Copy it unchanged to `lib/slack-bot/slack-bot.test.ts`, beside the modules it
imports. Its imports are relative to that path, so nothing in it needs adjusting.

## Manual end-to-end script

Against a real workspace, after deploy:

1. Install through the app's own Connect button, **not** Slack's config page.
   Confirm the consent screen lists "View email addresses".
2. Mention the bot in a channel. Expect the eyes reaction, then the hourglass,
   a streamed answer, and the hourglass removed.
3. Reply in that thread **without** mentioning it. Expect an answer.
4. Redeploy (forces cold instances), reply in the same thread again. Expect an
   answer. This is the shared-state check; with memory state it fails here.
5. DM the bot.
6. From a Slack account whose email has no app user, mention the bot. Expect the
   "couldn't match" message once; reply in a followed thread and expect silence.
7. Ask for an action that needs approval. Double-click **Approve** fast. Expect
   one side effect, buttons replaced by the result.
8. Have a colleague without access click a pending approval. Expect an ephemeral
   refusal visible only to them, and the request still pending.
9. Trigger an app-side report with a heading, a table and more than 3000
   characters. Expect a summary parent, the body in its thread, formatting intact.
