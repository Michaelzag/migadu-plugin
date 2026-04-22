---
name: dns-configuration
description: Translate Migadu's required DNS records (from get_domain_records) into the specific steps for the user's DNS provider — Cloudflare, Route 53, Namecheap, GoDaddy, Porkbun, Squarespace, etc. Use when the user needs to configure DNS after create_domain, when diagnostics are failing, or anywhere they're stuck in the registrar UI.
---

# Configuring DNS for a Migadu domain

DNS is the #1 stumbling block in Migadu onboarding. Users know what records they need (from `get_domain_records`), but translating those values into whatever registrar UI they're staring at is fiddly. This skill helps you walk them through it for their specific provider.

## Before you start

Call `get_domain_records(name="<their domain>")` first. You need the actual values — hostnames, priorities, key contents — before you can tell the user what to paste. Never hardcode values in your guidance; the DKIM key in particular is per-domain.

The response gives you:

| From the API     | Record type   | Where it goes                           |
|------------------|---------------|-----------------------------------------|
| `mx_records`     | MX            | Apex (`@`)                              |
| `spf`            | TXT           | Apex (`@`)                              |
| `dkim`           | TXT (or CNAME)| Subdomain (e.g. `key1._domainkey`)      |
| `dmarc`          | TXT           | `_dmarc` subdomain                      |
| `dns_verification`| TXT          | Usually apex or a specific subdomain    |

**Before you ask them to touch DNS, find out which provider they use.** Ask if you don't know: "Who do you have DNS with? Cloudflare, Route 53, Namecheap, something else?" The provider-specific guidance differs enough that you'll waste their time if you guess wrong.

## Apex notation

Different registrars represent "the domain itself" differently:

- Cloudflare, Namecheap, Porkbun, most modern UIs: `@`
- Route 53: leave the Name field empty (or set to the domain name itself)
- Some older UIs: the full domain name, e.g. `example.com`

For subdomains like `_dmarc` or `key1._domainkey`:
- Most UIs want just the subdomain part (e.g. `_dmarc`)
- A few want the FQDN (e.g. `_dmarc.example.com`)

When in doubt, enter just the subdomain part and the registrar will append the domain.

## Provider-specific notes

### Cloudflare

**The big gotcha**: Cloudflare's orange cloud (Proxy) must be **OFF** for mail-related records. Proxy routes through Cloudflare's servers, which makes MX resolution wrong and breaks mail delivery. Every mail record needs "DNS only" (grey cloud).

1. DNS → Records → Add record
2. MX: Name `@`, Mail server = value from API, Priority = from API, TTL auto, Proxy OFF
3. TXT records: Name from record type (`@`, `_dmarc`, `key1._domainkey`), Content = value (paste raw, Cloudflare handles quoting)
4. TTL: leave Auto

After saving, Cloudflare propagates within seconds, but upstream caches can still take 5-15 minutes.

### Route 53 (AWS)

1. Hosted zone → Create record
2. MX: Record name blank (apex), Value format `10 aspmx1.migadu.com.` (priority + hostname + trailing dot)
3. TXT: Record name as appropriate, Value = wrap in quotes (`"v=spf1 include:spf.migadu.com -all"`)
4. **Long TXT values (DKIM)**: Route 53 splits long values into multiple quoted strings automatically — usually fine, but if the DKIM check fails later, look at the record and verify the split happened at a sensible boundary

TTL: 300 (5 min) during setup, 3600 after confirmed working.

### Namecheap

1. Domain → Advanced DNS → Add new record
2. MX: Type MX, Host `@`, Value = hostname (no priority in value; priority is a separate field)
3. TXT: Type TXT, Host = `@` or subdomain, Value = paste raw (no quotes — Namecheap adds them)
4. TTL: Automatic is fine

Namecheap's email forwarding settings can conflict with external MX — if the user had email forwarding set up in Namecheap, disable it first.

### GoDaddy

Similar to Namecheap. "DNS" tab on the domain → Records.
- MX: Type MX, Name `@`, Value = hostname, Priority field separate, TTL 1 hour
- TXT: Type TXT, Name appropriate, Value = paste raw

GoDaddy sometimes hides advanced features behind a paid plan — DNS records themselves are always free, but the UI surface varies.

### Porkbun

Clean UI. DNS Records tab.
- Add Record: pick type, Host = `@` or subdomain, Answer = value, Priority (MX only), TTL default
- Porkbun handles TXT quoting cleanly. Paste values raw.

### Squarespace (formerly Google Domains)

Domain → DNS → Custom Records.
- Each record: Host, Type, Priority (MX), Data. Enter raw values.
- Apex is represented by `@`.

### IONOS, Hover, Gandi, others

Same pattern: find the DNS or nameservers page, add records by type with the appropriate host and value. If the user names a provider you don't recognize, ask them to screenshot the DNS page or paste the field labels — the mapping is almost always obvious once you see the column headers.

## Verifying manually

After the user saves the records, verify propagation with `dig` before calling `get_domain_diagnostics`. If the user has a shell handy:

```
dig MX example.com +short
dig TXT example.com +short                       # SPF
dig TXT _dmarc.example.com +short                 # DMARC
dig TXT key1._domainkey.example.com +short        # DKIM (and key2, key3...)
```

Or send them to mxtoolbox.com — paste the domain, it'll show all MX/SPF/DMARC/DKIM with clear pass/fail indicators.

## Common mistakes

- **MX pointing to the wrong host**: users sometimes type the Migadu web URL (`migadu.com`) as the MX target. It must be the specific MX host from `get_domain_records`.
- **Priority in the wrong field**: MX has priority + hostname as separate fields in most UIs. Users sometimes type `10 aspmx1.migadu.com` into the hostname field. Split them correctly.
- **Quotes in TXT values**: some registrars want quotes (`"v=spf1..."`), some don't. If the user pastes with quotes into a UI that auto-quotes, they'll get double-quoted garbage. Check what the registrar expects.
- **DKIM line-wrapped**: DKIM keys are long (250+ chars for 1024-bit, 450+ for 2048-bit). If the user's clipboard added line breaks, the key is broken. Paste as one continuous string; registrars that need splitting should handle it themselves.
- **Forgetting the underscore**: `_dmarc` and `_domainkey` start with an underscore. Easy to drop in a rushed copy-paste.
- **Trailing dot confusion**: Route 53 wants trailing dots on FQDNs in MX values; Cloudflare doesn't. If the record shows up in diagnostics but is "invalid", check the dot.
- **CAA records blocking**: if the user has CAA records restricting who can issue certs, Migadu might not be in the allowed list. CAA only affects TLS certificate issuance, not mail delivery, but Migadu may need a CAA entry for `letsencrypt.org` if issuing certs.
- **Apex CNAME**: some providers don't allow CNAME at the apex. Migadu doesn't require this for mail — DKIM uses TXT or CNAME on a subdomain, so this isn't usually an issue.

## When diagnostics still fail

If `get_domain_diagnostics` reports a record as missing or wrong after the user saved it:

1. **Wait 5-10 more minutes**. Propagation isn't instant.
2. **Have them dig from their own machine**: `dig TXT example.com +short`. If their local dig shows the record but Migadu's diagnostics don't, there's a propagation lag. If their local dig also shows nothing, the record didn't save correctly — back to the registrar UI.
3. **Compare byte-for-byte**: run `get_domain_records` again and diff the expected value against what their registrar actually saved. Extra whitespace, missing characters, quotes where there shouldn't be any — these are the usual suspects.
4. **If SPF is failing specifically**: the user may already have an SPF record from another service. You can't have two SPF records on the same domain — merge them into one: `v=spf1 include:spf.migadu.com include:otherservice.com -all`.
