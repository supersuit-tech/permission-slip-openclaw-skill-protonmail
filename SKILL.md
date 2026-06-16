---
name: permission-slip-openclaw-skill-protonmail
description: Check, search, read, and send email through a Proton Mail account connected to Permission Slip. Use when the user says things like "check my email", "any new mail?", "search my inbox for X", "read that message", or "reply to so-and-so" and their Proton Mail account is managed via Permission Slip.
---

# Proton Mail (via Permission Slip)

This skill lets you act on the user's Proton Mail mailbox by driving the
`permission-slip` CLI. You never talk to IMAP/SMTP or Proton directly — every
action goes through Permission Slip, which enforces the user's approval policy.

This skill contains no code of its own: it is a thin shim over the
`permission-slip` CLI. All authentication, the Proton Mail Bridge connection,
approval enforcement, and default-account selection live in Permission Slip and
its built-in `protonmail` connector.

## Invoking the CLI

Always invoke the CLI through `npx`:

```bash
npx @permission-slip/cli@latest <command> [args...]
```

Do **not** call a bare `permission-slip` binary — it is not guaranteed to be on
`PATH`, and assuming a path like `~/.openclaw/.../bin/permission-slip` will fail
with "Command not found". `npx @permission-slip/cli@latest` always resolves the
CLI (downloading/caching it on first use). Every command below — `whoami`,
`connectors`, `request`, `request-status` — must use this `npx` form.

## Defaults

- **"Check my email" → read the inbox, newest 50 messages.**
  Run `read_inbox` with `{"limit": 50}`. Folder defaults to `INBOX`.
- **Account selection:** omit `--instance`. Permission Slip auto-selects the
  user's *default* Proton Mail instance. Only pass `--instance` if the user
  explicitly names a second account.

## Preflight (run once per session, before the first action)

1. `npx @permission-slip/cli@latest whoami` — confirm this agent is registered.
   If not, tell the user to register
   (`npx @permission-slip/cli@latest register ...`) and stop.
2. `npx @permission-slip/cli@latest connectors` — confirm `protonmail` is
   available. If it's missing, the user hasn't connected a Proton Mail account
   yet; point them at the connector setup docs and stop.

## Intent -> action mapping

| User says | Action | Params |
|-----------|--------|--------|
| "check my email", "any new mail?" | `protonmail.read_inbox` | `{"limit": 50}` |
| "any *unread* mail?" | `protonmail.read_inbox` | `{"limit": 50, "unread_only": true}` |
| "search for X from Y" | `protonmail.search_emails` | per the connector's search schema |
| "read that one" / "open it" | `protonmail.read_email` | `{"folder": <folder>, "message_id": <uid>}` |
| "reply to it" | `protonmail.reply_email` | reply params (**needs approval**) |
| "send an email to ..." | `protonmail.send_email` | send params (**needs approval**) |
| "archive it" | `protonmail.archive_email` | `{"folder", "message_ids"}` (**needs approval**) |

## How to run an action

```bash
npx @permission-slip/cli@latest request \
  --action protonmail.read_inbox \
  --params '{"limit": 50}'
```

The CLI prints JSON. Two outcomes:

- **Auto-approved (reads):** the result includes the action output. Each message
  carries a stable `{folder, uid}` — use those as `message_id` + `folder` for
  any follow-up `read_email` / `archive_email`. Do **not** use sequence numbers
  or the RFC `message_id_header`.
- **Pending approval (send / reply / archive):** the CLI returns a request id in
  a `pending` state. Tell the user plainly: *"That needs your approval — I've
  sent the request to Permission Slip; I'll know once you approve it."* Then
  poll with `npx @permission-slip/cli@latest request-status <id>` and report the
  outcome. Never
  claim a send/reply succeeded until the status is approved **and** executed.

## Presenting results

**The JSON from the CLI is for you, not the user. NEVER show it to them.**
The CLI prints JSON (and the agent's own command invocations) as an
implementation detail. The user should never see raw JSON, the `npx ...`
commands you ran, request ids, `uid`s, `message_id_header`s, flags like
`\\Seen`, or any other machine-facing field. Always translate the JSON into a
short, human-readable summary.

For an inbox/search read, present one line per message with:
sender (name only), subject, read/unread, and a friendly relative time. Use a
couple of scannable emojis — 📧 per message, 🔵 unread / ⚪ read, 📎 attachment,
⭐ flagged/important — but keep it tasteful (one or two icons per line, not a
wall of emoji).

### Example

Given CLI output like:

```json
{"result":{"emails":[
  {"uid":436,"subject":"The hidden cost of building your own docs","from":["Daniel from Mintlify "],"date":"2026-06-16T10:00:31-04:00","flags":["\\Recent"]},
  {"uid":322,"subject":"RE: RE: RE: RE: Rental property","from":["Erika Acevedo-Rodriguez "],"date":"2026-06-12T15:54:21Z","flags":["\\Answered","\\Seen"]},
  {"uid":46,"subject":"Re: hosting this summer","from":["Chris Kelty "],"date":"2026-05-27T09:15:12-04:00","flags":["\\Seen"]}
],"total":3}}
```

Present it as:

> Here's your inbox — 3 messages:
>
> 📧 🔵 **Daniel from Mintlify** — The hidden cost of building your own docs · today 10:00 AM
> 📧 ⚪ **Erika Acevedo-Rodriguez** — RE: Rental property · Jun 12
> 📧 ⚪ **Chris Kelty** — Re: hosting this summer · May 27
>
> Want me to open any of these or reply to one?

Notes on the mapping:
- `\\Seen` → read (⚪); absence of `\\Seen` → unread (🔵).
- Clean up noisy subjects (collapse `RE: RE: RE:` to a single `RE:`).
- Turn the ISO timestamp into a friendly relative/local time.
- Keep the `{folder, uid}` for each message in your own working memory so you
  can act on follow-ups ("open it", "reply") — just don't surface them.

### Other cases

- If the inbox read returns empty, say so plainly — don't retry blindly.
- On error, surface the connector's message in plain language (e.g. "Proton Mail
  Bridge isn't running") and suggest the fix rather than dumping the raw error
  JSON or guessing.
- If a preflight check (`whoami`/`connectors`) fails, tell the user what's
  wrong and the fix — don't paste the raw "Command not found" / JSON output.

## Constraints

- Max `limit` is 50 — never request more.
- One concern per request. For "reply to all of these", make one request per
  message (or use bulk if the user wants a single approval prompt).
- Reads are low-risk; treat any write (send/reply/archive/delete/move) as
  approval-gated and communicate the wait.
