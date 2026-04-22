---
name: client-setup
description: Walk the user through configuring an email client (Apple Mail, Thunderbird, Outlook, iOS Mail, Android, mutt, etc.) for a Migadu mailbox. Use when the user asks about IMAP/POP3/SMTP settings, "how do I set up email on my phone/laptop", autoconfig issues, or can't get a specific client to connect.
---

# Setting up an email client for a Migadu mailbox

All Migadu mailboxes across all customer domains use the **same Migadu server hostnames** — the server settings don't change per customer or per domain. What changes is the username (always the full email address of the mailbox) and the password (set when the mailbox was created, or reset via `reset_mailbox_password`).

This confuses users because they expect settings like `mail.theirdomain.com` — there's no such thing. Always use Migadu's shared hostnames.

## The canonical settings

| Protocol     | Hostname           | Port | Security      | Purpose                               |
|--------------|--------------------|------|---------------|---------------------------------------|
| IMAP         | `imap.migadu.com`  | 993  | SSL/TLS       | Read/sync mail (recommended)          |
| POP3         | `pop.migadu.com`   | 995  | SSL/TLS       | Download-and-delete (legacy use case) |
| SMTP         | `smtp.migadu.com`  | 465  | SSL/TLS       | Send mail (recommended, implicit TLS) |
| SMTP         | `smtp.migadu.com`  | 587  | STARTTLS      | Send mail (fallback for older clients)|
| ManageSieve  | `smtp.migadu.com`  | 4190 | STARTTLS      | Server-side mail filters              |

**Authentication for all of the above**:
- Username: **full email address** (e.g., `alice@example.com`, NOT just `alice`)
- Password: the mailbox's password (NOT the Migadu API key — the API key is server-side only)

## If the user doesn't know their mailbox password

Two paths:

1. **They set it at creation time** — have them retrieve it from wherever they saved it.
2. **They don't have it** — use the MCP tool to reset it:
   ```
   reset_mailbox_password([{"target": "alice@example.com", "new_password": "<new-strong-password>"}])
   ```
   Then use the new password in the email client.

If the mailbox was created with `password_method: "invitation"`, the user should have received a setup email with a link to set their own password. If that email is lost, `reset_mailbox_password` creates a known password they can use.

## Autoconfiguration

Most modern clients (Apple Mail, Thunderbird, the Gmail app, iOS Mail) can auto-detect Migadu settings from just the email address. Try that first:

1. Enter the full email address
2. Enter the password
3. Let the client probe for settings

If autoconfig fails, fall through to manual entry with the table above. Autoconfig commonly fails when:
- The domain's MX records haven't propagated yet (see `domain-onboarding` skill)
- The client is Outlook (Microsoft's autodiscover is opinionated and sometimes doesn't like Migadu's format)
- The client is a corporate deployment with locked-down autoconfig

## Per-client quick reference

**Apple Mail (macOS, iOS)** — Usually autoconfigs. If not: "Add Other Mail Account" → "Mail account", enter email + password, then on the incoming server screen pick IMAP, host `imap.migadu.com`, username = full email. Outgoing server: `smtp.migadu.com`, same credentials.

**Thunderbird** — Autoconfigs reliably. Add account, enter name/email/password, Thunderbird detects Migadu.

**Outlook (desktop)** — Usually needs manual. Add account → "Advanced setup" → IMAP. Incoming: `imap.migadu.com` port 993 SSL/TLS. Outgoing: `smtp.migadu.com` port 465 SSL/TLS. **Important**: under "More settings" → "Outgoing Server", check "My outgoing server requires authentication" and "Use same settings as incoming mail server" — Outlook defaults to no SMTP auth, which fails silently.

**iOS Mail** — "Add Account" → "Other" → "Add Mail Account". Same settings. If autoconfig fails, it'll prompt for manual; use IMAP with the values above.

**Android Gmail app** — "Add account" → "Other". Pick IMAP. Use the values above.

**mutt / neomutt** — Use `imap://alice@example.com@imap.migadu.com:993` and `smtp://alice@example.com@smtp.migadu.com:465`. URL-encode the `@` in the username if your mutt version doesn't handle it: `alice%40example.com`.

## Common failure modes

- **"Username or password incorrect" but the password is right**: the user typed just the local part (`alice`) instead of the full email (`alice@example.com`). Migadu requires the full address.
- **SMTP "relay access denied" or "authentication required"**: client isn't authenticating on outgoing. Many clients default to no SMTP auth. Enable it and reuse the IMAP credentials.
- **TLS handshake errors on port 587**: the client is trying SSL/TLS (implicit) on 587, which uses STARTTLS. Either switch to port 465 (SSL/TLS) or change security to STARTTLS on 587.
- **Outlook says "the server isn't responding"**: often autodiscover looking at the wrong endpoint. Skip autoconfig and enter manually.
- **Connects fine but can't send**: almost always SMTP auth isn't enabled. Reopen account settings → outgoing server → enable authentication.

## After setup

If the user wants server-side filters (organize incoming mail, auto-reply from a specific sender, etc.), their client needs ManageSieve configured separately — most clients hide this under "Filters" or "Server-side rules" and will ask for `smtp.migadu.com:4190 STARTTLS`. Thunderbird has a Sieve extension; Apple Mail doesn't support sieve natively.

If they want to send from additional addresses (e.g., `alice@` AND `support@` from the same account), that's an **identity** — use the `create_identity` tool. The identity has its own password for SMTP authentication; the client then uses that alternate email + the identity's password to send as that address.
