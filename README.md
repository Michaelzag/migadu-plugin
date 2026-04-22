# Migadu Claude Code Plugin

A Claude Code plugin for managing [Migadu](https://migadu.com/) email hosting. Wraps the [`migadu-mcp`](https://pypi.org/project/migadu-mcp/) MCP server with three workflow skills that guide the model through the operations that are hardest to get right from tool descriptions alone.

## What you get

- The full [Migadu MCP server](https://github.com/Michaelzag/migadu-mcp) — 35 tools covering domains, mailboxes, identities, aliases, forwardings, rewrites. Installed automatically via `uvx`.
- Five skills loaded on demand:
  - **`domain-onboarding`** — create → DNS records → wait for propagation → diagnostics → activate → usage. Handles the multi-minute external wait correctly.
  - **`dns-configuration`** — translates Migadu's required records into the specific UI steps for Cloudflare, Route 53, Namecheap, GoDaddy, Porkbun, Squarespace, and others. Covers the Cloudflare-proxy gotcha, DKIM line-wrapping, SPF merging, and the common apex-notation confusions.
  - **`routing-decisions`** — choose between mailbox, alias, rewrite, and forwarding. They overlap; the wrong choice is hard to migrate away from.
  - **`bulk-safely`** — order-of-operations, partial-failure detection, destructive-batch patterns.
  - **`client-setup`** — IMAP/SMTP/POP3 settings for Apple Mail, Thunderbird, Outlook, iOS, Android, mutt. Migadu's shared server hostnames, full-email-as-username pattern, and the Outlook SMTP-auth gotcha.

## Install

Prerequisite: [`uvx`](https://docs.astral.sh/uv/) on your PATH.

Add this repo as a plugin marketplace and install:

```
/plugin marketplace add Michaelzag/migadu-plugin
/plugin install migadu@Michaelzag/migadu-plugin
```

Or if you've already added it via the Anthropic marketplace (once published there), just:

```
/plugin install migadu
```

## Configuration

Set these environment variables in your shell before starting Claude Code, or in the `env` block of your MCP client config:

| Variable          | Required                   | Description                                               |
|-------------------|----------------------------|-----------------------------------------------------------|
| `MIGADU_EMAIL`    | Yes                        | Admin email on your Migadu account                        |
| `MIGADU_API_KEY`  | Yes                        | [Migadu API key](https://admin.migadu.com/account/api/keys) |
| `MIGADU_DOMAIN`   | No                         | Default domain for tools that accept an optional `domain` |

The plugin's `.mcp.json` expects these to come through the environment — it doesn't embed credentials.

## Usage

Once installed, ask Claude Code things like:

- "Onboard a new domain acme.example for email hosting"
- "Set up `support@acme.example` as an alias to Alice and Bob"
- "Create mailboxes for these 12 new employees and send them invitation emails"
- "Delete the mailboxes for everyone who left last quarter"
- "How do I set up my new `alice@acme.example` mailbox in Apple Mail / Outlook / my phone?"

The relevant skill loads automatically based on what you're asking, and the MCP tools execute.

## Repo layout

```
migadu-plugin/
├── .claude-plugin/
│   └── plugin.json            # Plugin manifest
├── .mcp.json                  # Points at `uvx migadu-mcp` on PyPI
├── skills/
│   ├── domain-onboarding/SKILL.md
│   ├── dns-configuration/SKILL.md
│   ├── routing-decisions/SKILL.md
│   ├── bulk-safely/SKILL.md
│   └── client-setup/SKILL.md
└── README.md
```

The MCP server code itself lives in a separate repo ([Michaelzag/migadu-mcp](https://github.com/Michaelzag/migadu-mcp)) and is published to PyPI. This plugin is a thin wrapper over that.

## License

MIT — see [LICENSE](LICENSE).
