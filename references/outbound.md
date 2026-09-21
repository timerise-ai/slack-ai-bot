# Outbound: the app posts to Slack

The half of "two-way" that starts on the app's side: a scheduled job finishes,
an alert fires, a draft needs review. No Chat SDK involved; this is the Slack
Web API over `fetch`, with the workspace's token from `slack_installations`.

## Calling it

```ts
import { botHost } from "@/lib/slack-bot/host";
import { markdownToMrkdwn } from "@/lib/slack-bot/mrkdwn";
import { postReport } from "@/lib/slack-bot/outbound";
import { BOT_STRINGS } from "@/lib/slack-bot/strings";

export async function deliverToSlack(
  destination: { teamId: string; channelId: string },
  report: { title: string; summary: string | null; markdown: string },
): Promise<void> {
  const installation = await botHost.installations.get(destination.teamId);
  if (!installation) throw new Error(`Slack workspace ${destination.teamId} is not connected`);
  await postReport(installation.botToken, destination.channelId, {
    title: report.title,
    summary: report.summary ? markdownToMrkdwn(report.summary) : null,
    body: markdownToMrkdwn(report.markdown),
    truncationNote: BOT_STRINGS.reportTruncated,
  });
}
```

Store the destination as **`(team_id, channel_id)`**, never a channel id alone:
channel ids are only meaningful inside their workspace, and the team id is what
selects the token.

Whether delivery failure should fail the job is the caller's decision. The
source app treated it as fire-and-forget and only logged; that hides a revoked
token or an archived channel indefinitely. Prefer recording the last delivery
error where an operator can see it — [operations.md](operations.md).

The bot must be **in the channel** to post there (`not_in_channel` otherwise).
Invite it, or add the `chat:write.public` scope for public channels.

## Web API client, blocks, reports

```ts
// lib/slack-bot/outbound.ts
const SLACK_API_BASE = "https://slack.com/api";

/** Slack limits: 3000 chars per section text, 50 blocks per message, ~4000 chars of fallback text. */
const SECTION_MAX = 3000;
const MAX_BLOCKS = 50;
const FALLBACK_TEXT_MAX = 4000;

export type SlackBlock = Record<string, unknown>;

export class SlackApiError extends Error {
  constructor(
    readonly method: string,
    readonly code: string,
  ) {
    super(`Slack ${method} failed: ${code}`);
    this.name = "SlackApiError";
  }
}

interface SlackResponse {
  ok: boolean;
  error?: string;
  ts?: string;
  [key: string]: unknown;
}

/**
 * One Slack Web API call. Slack answers HTTP 200 with `ok: false` for most
 * failures, so both layers are checked. A 429 is retried once after Retry-After.
 */
export async function slackApi(
  token: string,
  method: string,
  body: Record<string, unknown>,
  fetchImpl: typeof fetch = fetch,
): Promise<SlackResponse> {
  for (let attempt = 0; ; attempt++) {
    const res = await fetchImpl(`${SLACK_API_BASE}/${method}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${token}`,
        // Without the charset Slack logs a missing_charset warning on every call.
        "Content-Type": "application/json; charset=utf-8",
      },
      body: JSON.stringify(body),
    });

    if (res.status === 429 && attempt === 0) {
      const waitSeconds = Number(res.headers.get("retry-after") ?? "1");
      await new Promise((r) => setTimeout(r, Math.min(waitSeconds, 10) * 1000));
      continue;
    }
    if (!res.ok) throw new SlackApiError(method, `http_${res.status}`);

    const data = (await res.json()) as SlackResponse;
    if (!data.ok) throw new SlackApiError(method, data.error ?? "unknown_error");
    return data;
  }
}

/** Splits at paragraph, then line, then hard boundary so no section exceeds `maxSize`. */
export function splitIntoChunks(text: string, maxSize: number): string[] {
  const chunks: string[] = [];
  let remaining = text;
  while (remaining.length > maxSize) {
    const window = remaining.slice(0, maxSize);
    const para = window.lastIndexOf("\n\n");
    const line = window.lastIndexOf("\n");
    const cut = para > 0 ? para : line > 0 ? line : maxSize;
    chunks.push(remaining.slice(0, cut));
    remaining = remaining.slice(cut).replace(/^\n+/, "");
  }
  if (remaining.length > 0) chunks.push(remaining);
  return chunks;
}

/**
 * Builds the blocks for a long mrkdwn text. When the text does not fit in one
 * message the last block says so; it is never cut silently.
 */
export function buildTextBlocks(
  text: string,
  options: { header?: string; truncationNote: string },
): SlackBlock[] {
  const blocks: SlackBlock[] = [];
  if (options.header) {
    blocks.push({
      type: "header",
      // Header blocks reject plain_text longer than 150 characters.
      text: { type: "plain_text", text: options.header.slice(0, 150), emoji: true },
    });
  }
  // One block is always reserved for the truncation note.
  const maxSections = MAX_BLOCKS - blocks.length - 1;
  const chunks = splitIntoChunks(text, SECTION_MAX);
  for (const chunk of chunks.slice(0, maxSections)) {
    blocks.push({ type: "section", text: { type: "mrkdwn", text: chunk } });
  }
  if (chunks.length > maxSections) {
    blocks.push({
      type: "context",
      elements: [{ type: "mrkdwn", text: options.truncationNote }],
    });
  }
  return blocks;
}

export interface PostOptions {
  threadTs?: string | null;
  fetchImpl?: typeof fetch;
}

/** Posts blocks. Returns the message ts, which is its id for threading and chat.update. */
export async function postBlocks(
  token: string,
  channelId: string,
  blocks: SlackBlock[],
  fallbackText: string,
  options: PostOptions = {},
): Promise<string> {
  const data = await slackApi(
    token,
    "chat.postMessage",
    {
      channel: channelId,
      // Shown in notifications and by screen readers. Required alongside blocks.
      text: fallbackText.slice(0, FALLBACK_TEXT_MAX),
      blocks,
      ...(options.threadTs ? { thread_ts: options.threadTs } : {}),
    },
    options.fetchImpl,
  );
  if (!data.ts) throw new SlackApiError("chat.postMessage", "missing_ts");
  return data.ts;
}

export async function updateBlocks(
  token: string,
  channelId: string,
  ts: string,
  blocks: SlackBlock[],
  fallbackText: string,
  fetchImpl?: typeof fetch,
): Promise<void> {
  await slackApi(
    token,
    "chat.update",
    { channel: channelId, ts, text: fallbackText.slice(0, FALLBACK_TEXT_MAX), blocks },
    fetchImpl,
  );
}

/**
 * App-initiated report: a short summary as the parent message, the full text
 * in its thread. Keeps the channel scannable while the detail stays one click away.
 */
export async function postReport(
  token: string,
  channelId: string,
  report: { title: string; summary: string | null; body: string; truncationNote: string },
  fetchImpl?: typeof fetch,
): Promise<string> {
  const { title, summary, body, truncationNote } = report;
  if (!summary) {
    const blocks = buildTextBlocks(body, { header: title, truncationNote });
    return postBlocks(token, channelId, blocks, body, { fetchImpl });
  }
  const parentTs = await postBlocks(
    token,
    channelId,
    buildTextBlocks(summary, { header: title, truncationNote }),
    summary,
    { fetchImpl },
  );
  await postBlocks(token, channelId, buildTextBlocks(body, { truncationNote }), body, {
    threadTs: parentTs,
    fetchImpl,
  });
  return parentTs;
}

/**
 * Answers a button click privately through the interaction's `response_url`.
 * Needs no token and no channel membership; valid for 30 minutes, 5 uses.
 */
export async function respondEphemeral(
  responseUrl: string,
  text: string,
  fetchImpl: typeof fetch = fetch,
): Promise<void> {
  await fetchImpl(responseUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json; charset=utf-8" },
    body: JSON.stringify({ response_type: "ephemeral", replace_original: false, text }),
  });
}
```

Limits behind the constants, and what Slack does past each:

| Limit | Value | Past it |
|---|---|---|
| Section `text` | 3000 chars | Whole message rejected: `invalid_blocks` |
| Blocks per message | 50 | `invalid_blocks` |
| Header `plain_text` | 150 chars | `invalid_blocks` |
| Top-level `text` | ~4000 chars recommended | Truncated by Slack |
| `chat.postMessage` | about 1 per second per channel | HTTP 429 with `Retry-After` |

One oversize block rejects the entire message, so the splitter is what stands
between a long report and no report at all. When 50 blocks are not enough the
last block says the content was cut; the reader is never left guessing.

**`ok: false` on HTTP 200** is how Slack reports nearly every failure:
`channel_not_found`, `not_in_channel`, `token_revoked`, `missing_scope`.
A client that only checks `res.ok` reports success for all of them.

The 429 retry is a single attempt capped at ten seconds. *The source app had no
retry; this is an addition.* A job that posts to many channels should still pace
itself rather than lean on it.

## Markdown to mrkdwn

Models write Markdown. Slack's `mrkdwn` is a different, smaller language:

| Markdown | mrkdwn |
|---|---|
| `**bold**` | `*bold*` |
| `*italic*` | `_italic_` |
| `~~strike~~` | `~strike~` |
| `[label](url)` | `<url\|label>` |
| `# Heading` | none; use bold |
| tables | none; use a code block |

```ts
// lib/slack-bot/mrkdwn.ts
/**
 * Converts model-style Markdown to Slack mrkdwn. Slack renders `*bold*`,
 * `_italic_`, `~strike~` and `<url|label>`; it has no headings and no tables.
 *
 * Bold and headings are parked behind \u0000 placeholders until the end.
 * Converting `**x**` to `*x*` in place lets the italic pass see a single-asterisk
 * pair and turn the bold text into `_x_`.
 */
export function markdownToMrkdwn(markdown: string): string {
  const parked: string[] = [];
  const park = (text: string): string => {
    parked.push(text);
    return `\u0000${parked.length - 1}\u0000`;
  };

  let out = markdown.replace(/```[\s\S]*?```/g, (m) => park(m));
  out = out.replace(/`[^`\n]+`/g, (m) => park(m));

  // Tables: Slack has none, so keep the alignment in a code block.
  out = wrapTables(out, park);

  // "* item" bullets, before any asterisk is read as emphasis.
  out = out.replace(/^(\s*)[*-](\s+\S)/gm, "$1•$2");
  out = out.replace(/^(?:-{3,}|\*{3,}|_{3,})\s*$/gm, "───────────────────");

  out = out.replace(/\[([^\]]+)\]\(([^)\s]+)\)/g, (_m, label: string, url: string) =>
    park(`<${url}|${label}>`),
  );

  // Headings become bold. Inner ** is stripped so "# **Title**" is not "**Title**".
  out = out.replace(/^#{1,6}\s+(.+?)\s*#*$/gm, (_m, title: string) =>
    park(`*${title.replace(/\*\*/g, "")}*`),
  );
  out = out.replace(/\*\*\*(.+?)\*\*\*/g, (_m, t: string) => park(`*_${t}_*`));
  out = out.replace(/\*\*(.+?)\*\*/g, (_m, t: string) => park(`*${t}*`));
  out = out.replace(/(?<![*\w])\*(?!\s)([^*\n]+?)(?<!\s)\*(?![*\w])/g, "_$1_");
  out = out.replace(/~~(.+?)~~/g, "~$1~");

  // Placeholders can nest (a link inside a heading), so restore until stable.
  const restore = /\u0000(\d+)\u0000/g;
  while (restore.test(out)) {
    out = out.replace(restore, (_m, i: string) => parked[Number(i)] ?? "");
  }
  return out;
}

function wrapTables(text: string, park: (t: string) => string): string {
  const out: string[] = [];
  let table: string[] = [];
  const flush = (): void => {
    if (table.length === 0) return;
    out.push(park("```\n" + table.join("\n") + "\n```"));
    table = [];
  };
  for (const line of text.split("\n")) {
    if (/^\s*\|.+\|\s*$/.test(line)) table.push(line.trim());
    else {
      flush();
      out.push(line);
    }
  }
  flush();
  return out.join("\n");
}
```

The placeholder parking is the point of this file. The source app's converter
rewrote in place, in sequence, and its passes fed each other. Running it shows:

| Input | Source output | Rendered in Slack as |
|---|---|---|
| `***both***` | `__both__` | literal underscores |
| `# **Title**` | `**Title**` | literal asterisks |
| `2 * 3 * 4` | `2 _ 3 _ 4` | arithmetic silently rewritten |

All three are regression tests in [testing.md](testing.md). Model output opens
with a bold heading often enough that the second one would show up in most
reports.

`chat.postMessage` also accepts a `markdown_text` field that Slack renders as
Markdown natively. It is capped at 12,000 characters and is mutually exclusive
with `text` and `blocks`, so it cannot carry a header block, a truncation note or
buttons. For a plain short message it replaces this converter; for reports and
approvals it does not. *Noted from the Slack adapter's source, not used by the
source app.*

This converter is for **app-generated** text going through `chat.postMessage`.
Replies streamed through the Chat SDK's `thread.post` are converted by the
adapter; do not run them through this as well.

## Strings

```ts
// lib/slack-bot/strings.ts
/**
 * Every user-visible string the bot posts. Slack messages render outside the
 * app's i18n runtime, so they live in one block: translate here, or swap the
 * object for a lookup keyed by the workspace's or user's locale.
 */
export const BOT_STRINGS = {
  identityNoEmail:
    "I can't read your Slack email, so I can't tell who you are. Ask a workspace admin to reinstall the app so it gets the email permission.",
  identityNoMatch:
    "I couldn't match your Slack account to a user here. Sign up with the same email address you use in Slack.",
  identityUnavailable: "I couldn't reach Slack to check who you are. Try again in a minute.",
  answerFailed: "Something went wrong while I was working on that. Nothing was changed; try again.",
  streamFallback: "I ran into a problem finishing that reply.",
  previewTruncated: "…(truncated)",
  approveButton: "Approve",
  cancelButton: "Cancel",
  approvalApproved: ":white_check_mark: *Approved and done*",
  approvalCancelled: ":no_entry_sign: *Cancelled*",
  approvalFailed: ":x: *Failed:*",
  approvalNotFound: "That request no longer exists.",
  approvalForbidden: "You don't have access to approve this.",
  approvalAlreadyDecided: "Someone already decided this one.",
  reportTruncated: "Content truncated. The full version is in the app.",
} as const;
```

Slack messages render outside the app's i18n runtime, so there is no provider to
hook into. One block of keys keeps them translatable; a multilingual product
swaps the object for `stringsFor(locale)` and passes the workspace's or user's
locale down.

## Checklist

- [ ] Destinations stored as `(team_id, channel_id)`
- [ ] Token looked up by team, never from env and never "first row"
- [ ] Both HTTP status and `ok` checked
- [ ] App-generated Markdown converted; SDK-streamed replies left alone
- [ ] Long text split under 3000 per section and 50 blocks, with a visible truncation note
- [ ] Every post has fallback `text`
- [ ] Bot invited to each destination channel
