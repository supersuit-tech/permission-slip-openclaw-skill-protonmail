# permission-slip-openclaw-skill-protonmail

An [OpenClaw](https://github.com/supersuit-tech) agent skill that lets an agent
check, search, read, and send **Proton Mail** email on the user's behalf —
entirely through the [Permission Slip](https://github.com/supersuit-tech/permission-slip)
CLI, so every action respects the user's approval policy.

With this skill installed, a user can simply say **"check my email"** and the
agent reads their Proton Mail inbox (newest 50) via the Proton Mail connector,
using whichever Proton Mail account is set as the default in Permission Slip.

## How it works

This skill contains **no code of its own**. It is a thin instruction layer over
the `permission-slip` CLI:

- Authentication, the Proton Mail Bridge (IMAP/SMTP) connection, approval
  enforcement, and default-account selection all live in Permission Slip and its
  built-in `protonmail` connector.
- The skill's only job is mapping natural language to the right CLI command
  (`permission-slip request --action protonmail.<action> ...`) and presenting
  the JSON result.

Reads (`read_inbox`, `search_emails`, `read_email`) are low-risk and typically
auto-approve. Writes (`send_email`, `reply_email`, `archive_email`) are
approval-gated: the agent submits the request and waits for the user to approve
it in Permission Slip before reporting success.

## Prerequisites

- An OpenClaw machine configured as an agent for a Permission Slip server.
- The `permission-slip` CLI installed and registered (`permission-slip whoami`
  should succeed).
- A connected Proton Mail account in Permission Slip. See the
  [Proton Mail connector setup guide](https://github.com/supersuit-tech/permission-slip/blob/main/docs/connectors/protonmail.md).

## Installation

Install the skill onto your OpenClaw machine when you're ready to use it — there
is no auto-install. Clone or copy `SKILL.md` into your agent's skills directory.

## See also

- [Permission Slip](https://github.com/supersuit-tech/permission-slip)
- [Proton Mail connector docs](https://github.com/supersuit-tech/permission-slip/blob/main/docs/connectors/protonmail.md)
- [permission-slip-openclaw-skill-gmail](https://github.com/supersuit-tech/permission-slip-openclaw-skill-gmail) — Gmail skill
- [permission-slip-openclaw-skill-google-calendar](https://github.com/supersuit-tech/permission-slip-openclaw-skill-google-calendar) — Google Calendar skill
- [permission-slip-openclaw-skill-google-drive](https://github.com/supersuit-tech/permission-slip-openclaw-skill-google-drive) — Google Drive skill
- [permission-slip-openclaw-skill-slack](https://github.com/supersuit-tech/permission-slip-openclaw-skill-slack) — Slack skill
