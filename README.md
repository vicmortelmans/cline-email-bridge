# Cline email bridge (first version)

A deliberately generic, one-shot IMAP-to-Cline bridge. Run it from cron; it does
not implement its own polling loop. Business-specific interpretation belongs in
Cline global/project rules and skills.

## What it does

- Searches the configured IMAP folder for unseen messages.
- Accepts only senders listed in `ALLOWED_SENDERS`.
- Saves the original `.eml`, extracted text/HTML bodies, and attachments under
  `TASKS_DIR`, outside the project.
- Starts one Cline CLI task per message, using the configured provider/model.
- Saves NDJSON output, stderr, final result, and metadata for each run.
- Marks successful messages as read and optionally moves them to a processed folder.
- Leaves failed messages unread. Failed tasks are not retried on every cron run
  unless `--retry-failed` is supplied.
- Uses a filesystem lock to prevent overlapping cron invocations.

It does not send a reply email yet. The final Cline response is stored as
`run-NNN/result.md`.

## Install

From the project directory:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Copy `.env.example` to `.env`, edit it, and ensure `.env` is in `.gitignore`.
If `.gitignore` already exists, add the snippet rather than replacing the file.

```bash
cp .env.example .env
chmod 600 .env
```

The script loads `.env` from beside itself by default. You can specify another
file with `--env-file /path/to/private.env`.

Make sure the Cline CLI is installed and authenticated/configured for the
provider in `CLINE_PROVIDER`. The script passes `--provider` and, if configured,
`--model` to the CLI. It uses the normal Cline data directory; it does not set
`--data-dir`, so session history should remain in the usual Cline location.

## Test safely

First, configure a trusted sender and run:

```bash
.venv/bin/python cline_email_bridge.py --dry-run
```

This saves unseen email and attachments but does not launch Cline or mark messages
as read. Inspect the resulting task directory.

Then run one message:

```bash
.venv/bin/python cline_email_bridge.py --limit 1
```

The first version defaults `CLINE_AUTO_APPROVE=false`. In unattended mode, Cline
may be unable to proceed when a tool requires approval. Set
`CLINE_AUTO_APPROVE=true` only after you have considered the consequences:
Cline may execute tools and commands without asking you. Use a dedicated,
appropriately restricted workspace and configure Cline rules/hooks before enabling it.

If a task fails, inspect its `run-NNN/stdout.jsonl`, `stderr.log`, and
`metadata.json`. To retry a failed message:

```bash
.venv/bin/python cline_email_bridge.py --retry-failed --limit 1
```

The original message remains unread after failure. If the message is marked
`rejected_sender`, correct `ALLOWED_SENDERS`; it is not automatically retried
until its metadata is removed or reset.

## Cron example

Run every five minutes. Use absolute paths; cron has a minimal environment.

```cron
*/5 * * * * /home/your-user/project/.venv/bin/python /home/your-user/project/cline_email_bridge.py >> /home/your-user/cline-email-bridge-cron.log 2>&1
```

The lock file prevents overlapping runs. The script processes up to
`MAX_EMAILS_PER_RUN` messages per invocation.

## Task directory layout

```text
TASKS_DIR/
  imap-UIDVALIDITY-UID/
    original.eml
    body.txt
    body.html
    metadata.json
    attachments/
    run-001/
      command.txt
      stdout.jsonl
      stderr.log
      result.md
```

The CLI prompt is intentionally generic. It tells Cline where the original
message and attachments are and asks it to apply the relevant standing rules
and skills. No business-specific classification is implemented in Python.

## Notes and limitations

- `ALLOWED_SENDERS` checks the email's From header; it is a useful basic guard,
  not cryptographic sender authentication. Use a dedicated mailbox and provider
  anti-spoofing protections; do not expose this mailbox publicly.
- The original email and attachments are untrusted input. Do not let their
  contents override your standing rules or security policies.
- This version processes `UNSEEN` messages. Do not manually mark a failed
  message as read unless you intend to remove it from automatic consideration.
- A process crash after Cline has started but before the message is marked read
  can lead to a repeat run on the next invocation. Check `metadata.json` and
  Cline history before retrying such cases.
- No SMTP reply is sent in this version; results are saved locally.
