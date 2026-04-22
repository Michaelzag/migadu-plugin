---
name: domain-onboarding
description: Onboard a new Migadu email domain end-to-end — create it, fetch DNS records, wait for propagation, run diagnostics, activate, and verify usage. Use when the user wants to add a new domain to their Migadu account or asks "how do I set up email for X.com".
---

# Onboarding a Migadu domain

Migadu domain setup is a six-step sequence with an external wait for DNS propagation in the middle. The Migadu API cannot activate a domain until external DNS records are published, so you cannot do this in one shot — plan on coming back after the user configures DNS at their registrar.

## The sequence

```
create_domain → get_domain_records → [user configures DNS externally]
             → get_domain_diagnostics (loop until pass)
             → activate_domain → get_domain_usage
```

## Step-by-step

### 1. `create_domain`

```
create_domain([{
  "name": "example.com",
  "hosted_dns": false,
  "create_default_addresses": true
}])
```

- **`hosted_dns: false`** is correct unless the user explicitly asks for Migadu-hosted DNS. Migadu themselves recommend external DNS.
- **`create_default_addresses: true`** creates `postmaster`, `abuse`, and `admin` aliases automatically. Useful for RFC compliance.
- On success, the domain exists in Migadu's system but is **not active yet**. No email flows until DNS is configured and `activate_domain` is called.

### 2. `get_domain_records`

Returns the DNS records the user must publish at their registrar. Response shape:

```json
{
  "domain_name": "example.com",
  "mx_records": [...],
  "spf": {...},
  "dkim": {...},
  "dmarc": {...},
  "dns_verification": {...}
}
```

Present these to the user clearly, one record set at a time. They need to configure **all** of them at their DNS provider (Cloudflare, Route 53, Namecheap, etc.). This is the manual step — Migadu cannot do this for them.

### 3. Wait for DNS propagation

After the user says they've added the records, wait at least 5 minutes before running diagnostics. Global DNS propagation typically takes 5-15 minutes but can stretch to hours depending on their provider and TTL settings.

### 4. `get_domain_diagnostics`

```
get_domain_diagnostics(name="example.com")
```

Returns a `checks` array with pass/fail status per record type. If anything is failing:
- MX not detected → user's MX records haven't propagated yet, or are wrong
- SPF/DKIM/DMARC missing → same, or incorrect values
- Verification record missing → same

Do not proceed to `activate_domain` until all checks pass. Wait another 5 minutes and re-run diagnostics. Repeat up to 3-4 times. If still failing after ~20 minutes, have the user double-check the record values at their DNS provider — propagation isn't the problem at that point.

### 5. `activate_domain`

```
activate_domain([{"name": "example.com"}])
```

Fails with HTTP 422 if DNS diagnostics are still failing. If you get 422, loop back to step 4. On success the domain is live and can receive/send email.

### 6. `get_domain_usage`

```
get_domain_usage(name="example.com")
```

Confirms the domain is tracking activity. Returns incoming/outgoing message counts and storage metrics. Useful to check at the end of onboarding and periodically for monitoring.

## After activation

Typical next steps the user will want:
- Create mailboxes with `create_mailbox` (see the `bulk-safely` skill if creating many)
- Set up aliases or rewrites (see `routing-decisions` skill)
- Create the first user mailbox (`admin@domain` already exists if `create_default_addresses: true`)

## Common failures

- **422 on activate_domain after diagnostics reported pass**: race condition with DNS TTLs. Wait 60s and retry.
- **create_domain returns 400 on the name**: domain already exists on another Migadu account, or the name is malformed.
- **`hosted_dns: true` was used by mistake**: Migadu plans to discontinue hosted DNS. If the domain was created with hosted_dns: true, best path is to delete via admin UI (no API delete) and recreate with `hosted_dns: false`.
