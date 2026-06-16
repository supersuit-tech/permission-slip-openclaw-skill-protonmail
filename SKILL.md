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

## Defaults

- **"Check my email" → read the inbox, newest 50 messages.**
  Run `read_inbox` with `{"limit": 50}`. Folder defaults to `INBOX`.
- **Account selection:** omit `--instance`. Permission Slip auto-selects the
  user's *default* Proton Mail instance. Only pass `--instance` if the user
  explicitly names a second account.

## Preflight (run once per session, before the first action)

1. `permission-slip whoami` — confirm this agent is registered. If not, tell
   the user to register (`permission-slip register ...`) and stop.
2. `permission-slip connectors` — confirm `protonmail` is available. If it's
   missing, the user hasn't connected a Proton Mail account yet; point them at
   the connector setup docs and stop.

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
permission-slip request \
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
  poll with `permission-slip request-status <id>` and report the outcome. Never
  claim a send/reply succeeded until the status is approved **and** executed.

## Presenting results

- Summarize the inbox briefly (sender, subject, unread/read, time). Don't dump
  raw JSON unless asked.
- If the inbox read returns empty, say so — don't retry blindly.
- On error, surface the connector's message verbatim (e.g. Bridge not running)
  and suggest the fix rather than guessing.

## Constraints

- Max `limit` is 50 — never request more.
- One concern per request. For "reply to all of these", make one request per
  message (or use bulk if the user wants a single approval prompt).
- Reads are low-risk; treat any write (send/reply/archive/delete/move) as
  approval-gated and communicate the wait.
