---
name: routing-decisions
description: Choose the right Migadu routing primitive — mailbox, alias, rewrite, or forwarding. Use when the user asks for anything involving email delivery to specific addresses, pattern-based routing, "forward X to Y", or anything that could reasonably be solved by more than one of these tools.
---

# Picking the right Migadu routing primitive

Migadu has four related but distinct routing mechanisms. They overlap enough to be confusing, and the wrong choice is hard to migrate away from. Decide up front.

## The decision tree

**Does the user need to store messages (read them in IMAP/POP3, persist on Migadu)?**
→ **Mailbox** (`create_mailbox`). Full account with auth, storage, protocol access. Everything else just routes.

**Are they forwarding to one or more addresses on the SAME Migadu domain, with no storage?**
→ **Alias** (`create_alias`). Single local-part → list of destinations. Constraint: destinations must be on the same domain.

**Are they forwarding to an EXTERNAL domain (different domain than the source)?**
→ **Forwarding** (`create_forwarding`). External delivery copy. **Requires confirmation** — Migadu sends a confirmation email to the destination; the forwarding sits in `pending` state until confirmed.

**Do they want wildcard/pattern matching (`demo-*@domain` → somewhere)?**
→ **Rewrite** (`create_rewrite`). Pattern-based routing with `local_part_rule`.

## Side-by-side

|                    | Mailbox      | Alias        | Forwarding    | Rewrite       |
|--------------------|--------------|--------------|---------------|---------------|
| Stores messages?   | Yes          | No           | No            | No            |
| Auth / IMAP?       | Yes          | No           | No            | No            |
| Same domain only?  | —            | Yes          | No (external) | Yes (sources) |
| Pattern matching?  | No           | No           | No            | Yes           |
| Confirmation flow? | No           | No           | Yes           | No            |
| Attached to?       | Domain       | Domain       | Mailbox       | Domain        |

## Concrete examples

**"`support@example.com` should go to Alice and Bob"**
→ Alias. Both destinations are on `example.com`. `create_alias` with `destinations=["alice@example.com", "bob@example.com"]`.

**"`alice@example.com` should also deliver a copy to `alice@gmail.com`"**
→ Forwarding. External destination. `create_forwarding` on the `alice` mailbox with `address=alice@gmail.com`. Alice (at gmail) will get a confirmation email she must click.

**"Anything matching `demo-*@example.com` should go to the sales team"**
→ Rewrite. Pattern-based. `create_rewrite` with `local_part_rule="demo-*"`, `destinations=["sales@example.com"]`.

**"Create an email account for Alice that she can log into with IMAP"**
→ Mailbox. She needs storage + auth. `create_mailbox`.

## Common mistakes

- **Using a mailbox when an alias would do.** If the user never logs in — never reads the inbox, never replies — they don't need a mailbox. Aliases are free and require no password management.
- **Using an alias for external forwarding.** Aliases can only target the same domain. For external destinations, use `create_forwarding` on a mailbox.
- **Creating many similar aliases instead of a rewrite.** If the user has `foo-support@`, `bar-support@`, `baz-support@` all going to the same place, that's a single rewrite rule: `*-support`, not three aliases.
- **Forgetting forwardings need confirmation.** The destination must click a link in the confirmation email before forwarding activates. If the user expects instant forwarding, warn them about the confirmation step.

## Multi-step patterns

Some setups combine primitives:

- **Catchall with exceptions**: alias `postmaster@` → admin, plus a rewrite `*-team@` → sales. Unmatched addresses fall through to the domain's catchall setting (configured via `update_domain`, not a separate tool).
- **Send-as another address on the same mailbox**: that's an **identity**, not any of the four above. Use `create_identity` on the mailbox. Identities share the mailbox's storage but let users send from additional addresses.
