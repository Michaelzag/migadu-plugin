---
name: bulk-safely
description: Run bulk Migadu mutations safely — ordering, partial-failure detection, and cleanup patterns. Use when the user asks to create, update, or delete more than a handful of mailboxes/aliases/identities/forwardings at once, or mentions anything that implies a batch ("onboard these 30 new hires", "delete everyone from the old team").
---

# Safe bulk operations against Migadu

Every mutation tool in the `migadu` MCP (`create_*`, `update_*`, `delete_*`, `activate_*`, `set_autoresponder`, `reset_mailbox_password`) accepts a `list[dict]` and returns a bulk-result envelope. They run items sequentially and capture per-item failures instead of aborting. That makes bulk ops resilient, but it also means **you must inspect the result, not just check for an exception**.

## Checking results correctly

```json
{
  "items": [
    {"mailbox": {...}, "email_address": "alice@x.com", "success": true},
    {"error": "password is required when password_method is password", "item": {...}, "success": false},
    ...
  ],
  "total_requested": 10,
  "total_successful": 9,
  "total_failed": 1,
  "success": false
}
```

- Top-level `success` is `true` only if **every** item succeeded. If it's `false`, look at `items[]` to find which ones failed and why.
- The `error` field on a failed item is a plain string. Common shapes: Pydantic validation errors (missing required field, invalid email) or Migadu API errors (`HTTP 400: ...`).
- Never report "done" based on the tool returning — read `total_successful` vs `total_requested`.

## Order of operations

Migadu resources have referential dependencies. Create in this order, delete in reverse:

```
Domain → Mailbox → Identity / Forwarding / Autoresponder
                 ↘ Alias, Rewrite (domain-level, no mailbox dependency)
```

- **Create**: domain must exist and be active before mailboxes. Mailboxes must exist before identities and forwardings on them.
- **Delete**: delete identities and forwardings before their parent mailbox, or the API rejects (or silently orphans) the child resources.
- **Update**: order usually doesn't matter, but if you're renaming a rewrite and updating its destinations in the same batch, do the rename first.

## Destructive batches

For any batch involving `delete_*`:

1. **List first.** Call the corresponding `list_*` tool and confirm the targets match what the user described. A typo in their filter could nuke the wrong accounts.
2. **Show the user the list.** "I'm about to delete these 14 mailboxes: [alice, bob, ...]. Confirm?" Wait for explicit confirmation.
3. **Run the delete.** Check `total_failed`. If any item failed, report which and stop before running further destructive ops.
4. **Verify.** Re-run `list_*` and confirm the targets are gone.

## Rollback patterns

Migadu has no transaction semantics — you cannot roll back a partial failure. Mitigations:

- **Dry-run style**: run `list_*` before and after so you have a diff.
- **Mirror before delete**: for delete batches, have the user export or note the resource details first (run `get_*` and save the output). If they need to restore, recreate from those details.
- **Small batches**: if doing 100+ items, break into chunks of 10-20 so a single failure doesn't taint the whole batch.

## Idempotency

- **Delete** is idempotent — calling `delete_mailbox` twice on the same target is safe. Migadu returns 500-with-success-shape on the first call (handled by the MCP client) and 404 on the second (also handled). The MCP reports both as success.
- **Create** is not idempotent — a second `create_mailbox` with the same `target` returns HTTP 400. If re-running, filter out existing resources first via `list_*`.
- **Update** is idempotent for scalar fields (setting `may_send=false` twice is fine). For `destinations` on aliases/rewrites, update replaces the whole list — not append.

## Common pitfalls

- **Batch creates that depend on each other in the same call**: don't put mailbox creation and identity creation on that mailbox in the same `create_*` call. Each tool handles one resource type. Order them across separate tool calls.
- **Assuming `total_successful > 0` means "partially ok, continue"**: for linked resources, a partial failure probably means you need to roll forward carefully. If 8 of 10 mailboxes created successfully but 2 failed, do NOT try to create identities for all 10 — create identities only for the 8 that succeeded.
- **Ignoring Pydantic validation errors**: these show up in `items[].error` as strings like `"Validation failed: password: field required"`. They're user-input errors, not API errors. Fix the input and retry — don't escalate to the user as if Migadu is down.
