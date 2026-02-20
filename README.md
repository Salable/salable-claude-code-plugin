# Salable Plugin for Claude Code

Monetize any app with [Salable 2.0](https://beta.salable.app) directly from Claude Code. This plugin provides MCP tools for catalog provisioning and guided workflows for building pricing pages, entitlement gating, checkout flows, and subscription management into your app.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed
- A [Salable](https://beta.salable.app) account with an API key

## Setup

### 1. Add the marketplace and install the plugin

```bash
claude plugin marketplace add Salable/salable-claude-code-plugin
claude plugin install salable
```

### 2. Set your API key

```bash
export SALABLE_API_KEY="your_api_key_here"
```

Add it to your shell profile (`~/.zshrc` or `~/.bashrc`) for persistence:

```bash
echo 'export SALABLE_API_KEY="your_api_key_here"' >> ~/.zshrc
source ~/.zshrc
```

### 3. Restart Claude Code

The MCP server connection requires a restart after setting the API key.

## Usage

Run the `/salable-monetize` skill inside Claude Code:

```
/salable-monetize
```

This launches a guided workflow that:

1. **Discovers your app** — reads your codebase to understand the stack, auth setup, and existing integrations
2. **Plans your monetization** — walks through features, entitlements, plans, pricing, and billing intervals with you
3. **Provisions the catalog** — creates products, entitlements, plans, and pricing via Salable MCP tools
4. **Builds app integration** — generates pricing pages, entitlement checks, checkout flows, and subscription management views using the Salable REST API

### What it builds

- **Public pricing page** with plan comparison and checkout buttons
- **Entitlement-based feature gating** with a client-side React context provider
- **Checkout flow** via Salable cart API and Stripe
- **Subscription management page** with plan status, billing portal, and invoice access
- **API routes** for entitlements, checkout, and subscription management
- **Navigation wiring** and auth protection

### Supported pricing models

| Model | Description |
|---|---|
| Flat-rate | Fixed monthly/yearly subscription fee |
| Per-seat | Price per team member with configurable seat limits |
| Metered | Usage-based billing with graduated or volume tiers |
| Hybrid | Any combination of the above in a single plan |

Multiple billing cadences (monthly, quarterly, yearly) and multi-currency pricing are supported.

## Plugin structure

```
salable-claude-code-plugin/          # repo root
├── .claude-plugin/
│   └── marketplace.json             # Plugin collection manifest
├── plugins/
│   └── salable/
│       ├── .claude-plugin/
│       │   └── plugin.json          # Plugin config and MCP server definition
│       ├── .mcp.json                # MCP server config
│       ├── skills/
│       │   └── salable-monetize/
│       │       └── SKILL.md         # /salable-monetize skill definition and workflow
│       └── references/
│           ├── auth-options.md          # Auth recommendations by stack
│           ├── mcp-tool-playbook.md     # MCP operation guide
│           ├── openapi-focus.md         # REST API endpoint map
│           └── pricing-model-templates.md  # Ready-to-use pricing payloads
├── README.md
└── .gitignore
```

## How it works

The plugin connects Claude Code to the Salable platform through two channels:

- **MCP tools** (`mcp__salable__*`) for catalog writes — creating products, entitlements, plans, line items, and prices
- **REST API** (`beta.salable.app/api/*`) for app runtime — pricing pages, entitlement checks, checkout, and subscription management

The `/salable-monetize` skill orchestrates both, using MCP for provisioning and REST for the code it generates in your app.

## Requirements

- Your app must have authentication before monetization can be integrated. The plugin checks for this and will recommend auth solutions for your stack if none is found.
- Identity fields (`owner`, `grantee`) must use non-email identifiers (org IDs, user IDs, or deterministic hashed values).

## Links

- [Salable 2.0 Dashboard](https://beta.salable.app)
- [Salable Docs](https://beta.salable.app/docs)
- [Salable OpenAPI Spec](https://beta.salable.app/openapi.yaml)
