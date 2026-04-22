# Migadu Claude Code Plugin

Claude Code plugin for [Migadu](https://migadu.com/) email hosting. Manages domains, mailboxes, aliases, identities, forwardings, and rewrites — and knows how to walk you through the parts that are annoying to get right: DNS setup at your registrar, picking between routing primitives that overlap, configuring an email client, running bulk operations without trashing data.

The plugin bundles the [`migadu-mcp`](https://pypi.org/project/migadu-mcp/) MCP server (35 tools across the full Migadu API) with five skills that kick in when the tool descriptions alone aren't enough.

## Install

Prerequisite: [`uvx`](https://docs.astral.sh/uv/) on your `PATH`.

```
/plugin marketplace add Michaelzag/migadu-plugin
/plugin install migadu@michaelzag
```

Then set your credentials in the environment where Claude Code runs:

```bash
export MIGADU_EMAIL=you@example.com
export MIGADU_API_KEY=...       # https://admin.migadu.com/account/api/keys
export MIGADU_DOMAIN=example.com # optional — default domain for tools that accept one
```

Restart Claude Code and the `migadu` MCP server will show up under `/mcp` as Connected.

## What the skills do

Each skill loads on demand when the topic comes up in conversation. Try asking Claude Code the examples below; the relevant skill will activate automatically.

| Skill | What it covers | Try asking |
|-------|----------------|------------|
| **`domain-onboarding`** | Full domain setup: create → DNS records → wait for propagation → diagnostics → activate → usage. Handles the multi-minute external DNS wait. | *"Onboard `acme.example` for email hosting"* |
| **`dns-configuration`** | Registrar-specific record setup for Cloudflare, Route 53, Namecheap, GoDaddy, Porkbun, Squarespace, and others. Covers the Cloudflare-proxy gotcha, DKIM line-wrapping, SPF merging. | *"I added the DNS records but diagnostics still fail"* |
| **`routing-decisions`** | Pick between mailbox, alias, rewrite, and forwarding. They overlap — the wrong choice is hard to migrate away from. | *"`support@acme.example` should go to both Alice and Bob"* |
| **`bulk-safely`** | Order-of-operations, partial-failure detection, destructive-batch patterns, rollback guidance. | *"Create mailboxes for 40 new hires from this CSV"* |
| **`client-setup`** | IMAP/SMTP/POP3 settings for Apple Mail, Thunderbird, Outlook, iOS, Android, mutt. Covers Migadu's shared server hostnames, full-email-as-username pattern, and the Outlook SMTP-auth gotcha. | *"How do I set up `alice@acme.example` in Outlook?"* |

## What the MCP server exposes

35 tools across six resource types — domains, mailboxes, identities, aliases, forwardings, rewrites. Every mutation tool takes a `list[dict]` of items and returns a bulk-result envelope with per-item success/failure, so "create 40 mailboxes" and "create 1 mailbox" are the same call shape.

Read operations are summarized when they'd blow past a rough token budget, so `list_mailboxes` on a 500-mailbox domain returns counts plus a sample instead of flooding context.

Ten read-only MCP resource URIs are registered too (`domains://`, `mailbox://{domain}/{local_part}`, `domain-records://{name}`, and so on) for AI clients that prefer addressing data over tool calls.

Full tool inventory lives in [the MCP server repo](https://github.com/Michaelzag/migadu-mcp#tools).

## Configuration

| Variable | Required | Notes |
|----------|----------|-------|
| `MIGADU_EMAIL` | Yes | Admin email on your Migadu account |
| `MIGADU_API_KEY` | Yes | From [Migadu Admin → API Keys](https://admin.migadu.com/account/api/keys) |
| `MIGADU_DOMAIN` | No | Default domain for tools that accept an optional `domain` arg |

The plugin's `.mcp.json` expects these to come through the environment — credentials are never embedded in the plugin itself.

If you manage multiple Migadu domains, leave `MIGADU_DOMAIN` unset and pass `domain` explicitly on each tool call. Start with `list_domains` to see what's available on your account.

## Troubleshooting

**MCP server shows "Failed to connect" in `/mcp`.** Almost always missing env vars. Verify `MIGADU_EMAIL` and `MIGADU_API_KEY` are set in the environment Claude Code inherited. Run `echo $MIGADU_EMAIL` in the same shell where you launched Claude Code.

**"The token is invalid or expired."** The `MIGADU_EMAIL` must match the admin email tied to the API key. API keys are scoped to a specific admin user, not the organization.

**`uvx: command not found`.** Install [`uv`](https://docs.astral.sh/uv/). It ships `uvx` alongside `uv`.

**Skills don't fire when I expect them to.** Skills activate based on the conversation content matching their descriptions. If you want to force one, invoke it directly: `/migadu:domain-onboarding` (or whichever). The plugin's skills are listed under `/plugin` once installed.

**Destructive operations (delete mailbox/alias/etc.) report as failures but the resource is actually gone.** Was a real bug in the underlying MCP server pre-v3.6.0 — Migadu returns HTTP 500 on successful DELETEs and the old client mis-handled it. Fixed in `migadu-mcp>=3.6.0`. Run `uvx --refresh-package migadu-mcp migadu-mcp@latest` to pull the fix.

## How it fits together

```
Claude Code
    │
    ├── Skills (markdown, loaded on demand)
    │     domain-onboarding, dns-configuration,
    │     routing-decisions, bulk-safely, client-setup
    │
    └── MCP server (spawned via uvx)
          migadu-mcp on PyPI
          ├── 35 tools
          ├── 10 resource URIs
          └── 3 workflow prompts
```

Skills provide decision-making guidance and workflow structure; the MCP server provides the actual Migadu API access. The two are separate repos so the MCP server can be used standalone (`uvx migadu-mcp`) without the plugin layer if someone prefers.

MCP server: [Michaelzag/migadu-mcp](https://github.com/Michaelzag/migadu-mcp) on PyPI.

## License

MIT — see [LICENSE](LICENSE).
