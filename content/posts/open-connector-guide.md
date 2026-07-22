---
title: "Open Connector: The Open-Source Gateway That Lets Your AI Agents Use 1,000+ SaaS Tools (Setup Guide + 3 Real Workflows)"
date: 2026-07-22T08:00:00+08:00
draft: false
categories: ["Artificial Intelligence"]
tags: ["AI tools", "productivity"]
description: "Every developer building AI agents hits the same wall: your agent needs to read Gmail, create Notion pages, post Slack messages, open GitHub PRs, and query"
---

Every developer building AI agents hits the same wall: your agent needs to read Gmail, create Notion pages, post Slack messages, open GitHub PRs, and query Airtable — but handing raw OAuth tokens or API keys to an agent process is a security nightmare. You either bake credentials into environment variables (bad), build a custom auth proxy (tedious), or pay $99+/month for a closed-source solution.

Enter **Open Connector** — an open-source gateway that sits between your AI agents and 1,000+ SaaS tools. It handles OAuth flows, token refresh, credential storage, and access control so your agent calls a simple local endpoint instead of juggling API keys. It reached 3,095 GitHub stars in 23 days, has zero setup guides anywhere on the web — and it's completely free to self-host.

In this guide, you'll deploy Open Connector, connect your first SaaS tools, and build three real workflows: an agent that reads Gmail and drafts replies, an agent that queries Notion from Claude Code via MCP, and an agent that triggers GitHub Actions and reads BigQuery. Let's get started.

## What Is Open Connector?

Open Connector is an open-source, self-hostable API gateway built by the OOMOL Lab team. It standardizes how AI agents authenticate to and interact with external SaaS platforms. Instead of managing 20 different API clients and credential stores, you run one gateway service that:

- **Manages OAuth 2.0 flows** for 1,000+ providers (Gmail, Slack, GitHub, Notion, Airtable, Salesforce, Google Sheets, BigQuery, etc.)
- **Stores and refreshes tokens** securely — credentials never leave your infrastructure
- **Exposes a unified API** so your agent calls `http://localhost:3030/api/v1/tools/gmail/send` instead of building a Gmail API client
- **Supports MCP (Model Context Protocol)** — meaning Claude, Cursor, and any MCP-compatible agent can discover and call these tools automatically
- **Works everywhere** — deploy via Docker, Node.js, or Cloudflare Workers

The architecture is simple: a control plane (web UI + API) that manages connections, and a relay layer (oo-cli) that proxies requests from your local agent to the gateway. You authenticate once per provider, and every agent on your machine — or your team — can use those connections without seeing the credentials.

## Open Connector vs Composio — Comparison Table

| Feature | Open Connector (Free, Open-Source) | Composio (Closed-Source) |
|---|---|---|
| **Pricing** | Free (self-hosted) | $99/month (Pro) — $499/month (Team) |
| **License** | Apache 2.0 (open source) | Proprietary |
| **Provider count** | 1,000+ | 200+ |
| **Self-hosting** | Docker, Node.js, Cloudflare Workers | Not available (cloud-only) |
| **MCP support** | Native (tools auto-exposed via MCP) | Beta (limited) |
| **OAuth flow** | Built-in redirect + CLI auth | Built-in (cloud-managed) |
| **Token storage** | Your infrastructure (PostgreSQL/SQLite/Cloudflare KV) | Composio cloud |
| **Team sharing** | Yes (organization accounts, shared connections) | Yes (team plan) |
| **Local relay** | oo-cli (lightweight Go binary) | Proprietary SDK |
| **Authentication** | GitHub OAuth + API keys | API keys only |
| **Rate limiting** | Configurable per connection | Plan-based limits |
| **Audit logging** | Built-in via SQLite | Available on Team plan |

Open Connector wins on cost, provider catalog size, and data sovereignty. Composio wins if you want zero-infrastructure cloud management. For any team that already has Docker infrastructure or cares about credentials not leaving their network, Open Connector is the obvious choice.

## Step-by-Step: Deploy Open Connector and Connect Your First Tools

### 1. Deploy the Gateway (Docker, 2 minutes)

```bash
# Clone the repo
git clone https://github.com/oomol-lab/open-connector.git
cd open-connector

# Start with Docker Compose
docker compose up -d
```

This starts the gateway on `http://localhost:3030` with a PostgreSQL backend and the admin web UI. For a lighter setup, you can use SQLite:

```bash
docker compose -f docker-compose.sqlite.yml up -d
```

Open `http://localhost:3030` in your browser. You'll see the Open Connector dashboard.

### 2. Create Your First Connection (Gmail)

1. Click **New Connection** in the dashboard
2. Search for "Gmail" in the provider catalog
3. Click **Connect** — Open Connector redirects you to Google's OAuth consent screen
4. Authorize the scopes (read/write email)
5. You're redirected back to the dashboard. Gmail is connected.

That's it. The gateway now stores a refreshed OAuth token. Your agents never see the raw credential.

### 3. Install the Local CLI Relay

```bash
# Install oo-cli (macOS / Linux)
curl -fsSL https://get.openconnector.dev | sh

# Start the relay
oo-cli start --port 3030
```

The relay creates a local channel between your agent processes and the gateway. Any tool call to `localhost:3030` is authenticated, proxied, and routed to the right provider.

### 4. Connect More Tools

Repeat step 2 for each tool you need:

- **Slack** — select "Slack", authorize bot scopes (chat:write, channels:read)
- **GitHub** — select "GitHub", authorize repo and actions scopes
- **Notion** — select "Notion", authorize read/write content
- **Airtable** — select "Airtable", authorize data read/write
- **BigQuery** — select "Google BigQuery", authorize query scope

All connections appear in your dashboard. You can test each one with the built-in API playground.

### 5. Configure MCP (Optional — For Claude Code, Cursor, etc.)

If you use Claude Code, Cursor, or any MCP-compatible agent tool, add this to your MCP config:

```json
{
  "mcpServers": {
    "open-connector": {
      "command": "oo-cli",
      "args": ["mcp", "--port", "3030"]
    }
  }
}
```

Now your agent auto-discovers all connected tools. Ask Claude Code to "Find my latest Gmail from John and create a Notion page with a summary" — it handles the tool calls transparently.

## 3 Real Workflows You Can Build Today

### Workflow 1: Agent Reads Gmail → Drafts Reply (Claude Code + oo-cli)

With Gmail connected and MCP configured, run:

```bash
claude
> Check my inbox for unread emails from Sarah about the Q3 budget. Draft a reply summarizing the key points.
```

Behind the scenes:
1. Claude calls `gmail_list_messages` with `q=from:sarah is:unread`
2. Open Connector proxies to Gmail API, returns message list
3. Claude calls `gmail_get_message` for the email body
4. Claude analyzes the content and calls `gmail_draft_create` with a contextual reply
5. The draft appears in your Gmail Drafts folder — review and send

No Gmail API client code. No OAuth token refresh logic. No scopes to configure beyond the initial OAuth grant.

### Workflow 2: Agent Queries Notion Database from Claude Code

Connect Notion in the dashboard, then:

```bash
claude
> Look up all tasks in my Notion "Product Roadmap" database that are due this week and have status "In Progress"
```

The MCP bridge exposes Notion as discoverable tools: `notion_query_database`, `notion_get_page`, `notion_create_page`. Your agent queries, reads, and writes Notion without writing a single line of API integration code.

For programmatic use, hit the REST API directly:

```bash
curl -s http://localhost:3030/api/v1/tools/notion/query-database   -H "Content-Type: application/json"   -d '{"database_id": "YOUR_DB_ID", "filter": {"property": "Status", "select": {"equals": "In Progress"}}}'
```

### Workflow 3: Agent Triggers GitHub Actions + Checks BigQuery

This workflow shows multi-tool orchestration — an agent that deploys infrastructure and verifies it:

```bash
claude
> Trigger the "deploy-prod" GitHub Actions workflow on my repo, then wait for it to complete and check BigQuery for the deployment metrics in the last hour.
```

1. Claude calls `github_actions_create_workflow_dispatch` with the repo and workflow ID
2. Claude polls `github_actions_list_workflow_runs` until status is "completed"
3. On success, Claude calls `bigquery_jobs_query` with a SQL query for deployment metrics
4. Claude returns a summary: "Deployment completed at 14:32. 2,341 requests served, 0 errors, p99 latency 142ms."

This is a pattern you'd normally script with GitHub's REST API + `bq` CLI + JSON parsing. With Open Connector, your agent does it in one conversation.

## Best For / Worst For

**Best for:**
- Indie hackers and solo developers who want free agent-tool integration without subscription fees
- Teams that have existing Docker or Cloudflare infrastructure and want to self-host
- Anyone building agentic apps with Claude Code, Cursor, or custom MCP clients
- Developers who need 100+ tool integrations and don't want 100+ API client libraries
- Security-conscious teams that don't want credentials stored on third-party cloud services

**Worst for:**
- Non-technical users who want a fully managed, zero-setup solution (Composio or Zapier are better fits)
- Teams that prefer a single vendor SLA with guaranteed uptime and support
- Developers targeting only 1-2 tools who don't need a gateway abstraction
- Organizations with strict compliance requirements needing a SOC 2 certified provider

## Pricing

| Deployment | Cost | What You Get |
|---|---|---|
| Docker (SQLite) | $0/mo | Local PostgreSQL-less setup, single-user |
| Docker (PostgreSQL) | $0/mo + server cost | Multi-user, audit logs, team sharing |
| Cloudflare Workers | $0/mo (within Free tier limits) | 100k requests/day, global edge deployment |
| Self-hosted (any infra) | $0/mo | Unlimited connections, unlimited providers |
| Composio Pro (alternative) | $99/mo | Managed cloud, 200+ providers, monthly limits |
| Composio Team | $499/mo | Managed cloud, team accounts, priority support |

**Bottom line:** Open Connector costs infrastructure only. A $6/month VPS runs it comfortably for a small team. [AFFILIATE: DigitalOcean] — their $6/mo droplet is more than sufficient for a team of 5.

## FAQ

**Q: Can Open Connector handle rate limits from SaaS providers?**
A: Yes. It supports configurable rate limiting per connection. You can set max requests per minute for each provider, and queued requests wait for the next available window.

**Q: What happens when my OAuth token expires?**
A: Open Connector automatically refreshes tokens using the provider's refresh token flow. You never need to re-authorize unless you revoke the app on the provider side.

**Q: Is Open Connector compatible with any LLM provider?**
A: Yes. The REST API works with any HTTP-capable agent (OpenAI, Anthropic, open-source models). MCP support adds auto-discovery for Claude, Cursor, and any MCP-native agent.

**Q: Can multiple agents share the same connection pool?**
A: Absolutely. All connections are stored in the gateway. Any agent on your network that can reach `localhost:3030` (or your gateway address) can use them. oo-cli handles local proxying.

**Q: Does it support custom/private APIs?**
A: Yes. You can add custom providers by defining the OAuth flow and API schema. The provider catalog is open-source — you can submit PRs for any missing provider.

## Conclusion

Open Connector solves the most painful part of building agentic applications: connecting your agents to the tools they need without creating a credential management nightmare. It's free, open-source, self-hostable, supports 1,000+ providers, and works with any AI agent that speaks HTTP or MCP.

The three workflows you just built — Gmail reading and reply drafting, Notion database queries, and GitHub-to-BigQuery deployment verification — are patterns you'll reuse daily once your gateway is running. Every new tool you connect compounds the capability of every agent you run.

**Your next step:** Deploy Open Connector on your infrastructure this week. Connect GitHub, Gmail, and Slack first — those three unlock 90% of the useful agent workflows. After that, add Notion or Airtable for knowledge management. [LINK: Setting up agent workflows with MCP]

Once your gateway is live, every agent you build — CLI scripts, chatbot backends, automated workflows — gains instant access to the full provider catalog. That's the unlock. One gateway, 1,000+ tools, zero subscription fees.

---

*Not financial advice. Prices and availability as of July 2026. Open Connector is Apache 2.0 licensed — contributions welcome at [github.com/oomol-lab/open-connector](https://github.com/oomol-lab/open-connector).*
