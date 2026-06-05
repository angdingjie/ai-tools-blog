---
title: "Paste MCP Guide: How to Turn Your Clipboard Into AI Memory"
date: 2026-06-03T08:00:00+08:00
draft: false
categories: ["Artificial Intelligence"]
tags: ["AI tools", "productivity"]
description: "You copy links, code snippets, research notes, and screenshots dozens of times a day. But your AI tools — Claude, Codex, Cursor — can't see any of it. "
---

You copy links, code snippets, research notes, and screenshots dozens of times a day. But your AI tools — Claude, Codex, Cursor — can't see any of it. Every time you start a new conversation, you're re-typing or re-pasting context your clipboard already holds. That's the gap Paste MCP closes.

Launched June 2, 2026, Paste MCP is the first clipboard manager to expose your full clipboard history as live context for AI tools via the Model Context Protocol (MCP). Instead of manually feeding your AI tool piece by piece, Paste MCP lets you ask your AI to *find* what you copied — yesterday, this morning, across projects — and work with it directly. This guide walks you through exactly what Paste MCP does, how it compares to the manual approach, a step-by-step setup you can complete in 5 minutes, and the security considerations you need to know before connecting your clipboard to AI.

## What Is Paste MCP?

Paste is a clipboard manager for Mac, iPhone, and iPad that stores everything you copy — text, links, images, code snippets, even screenshots — in a searchable, organized history. It syncs across your Apple devices via iCloud and has built-in privacy controls like app-specific exclusion rules.

MCP (Model Context Protocol) is an open standard originally proposed by Anthropic and now stewarded by the Linux Foundation's Agentic AI Foundation. It gives AI tools a standardized way to read data from external sources — databases, file systems, and in this case, your clipboard history. It's the same protocol that connects tools like Zapier, Notion, and Figma to AI assistants.

Paste MCP combines the two. It runs a lightweight MCP server locally on your Mac that exposes your clipboard history to any MCP-compatible AI tool: Claude Desktop, OpenAI Codex, Cursor, and others. When your AI tool needs context, it can query Paste's history and pull relevant items — without you manually copy-pasting each one.

The key insight: **your clipboard is already your most active memory layer.** Every link you save for later, every snippet you grab from a research session, every screenshot you take for reference — it's all sitting in your clipboard history. Paste MCP makes that history addressable by AI, turning passive storage into active context.

## Paste MCP vs. Manual Copy-Paste — Comparison Table

| Capability | Manual Copy-Paste | Paste MCP |
|---|---|---|
| **Context depth** | Last item copied only (single clipboard slot) | Full searchable history across days/weeks |
| **Multi-item retrieval** | Must switch windows and paste each item one at a time | Ask AI to "find everything I copied about project X this week" |
| **Cross-session memory** | Lost when you close the chat | Persistent — clipboard history is always available |
| **Media support** | Text only (most AI tools) | Text, links, screenshots, file references |
| **Setup time** | Zero (but zero capability) | ~5 minutes (one-time MCP configuration) |
| **Search functionality** | Manual scroll through tabs or notes | AI-powered semantic search across clip history |
| **Security control** | No clipboard access (but no AI access either) | Granular: enable/disable per tool, exclude sensitive apps |
| **Workflow continuity** | Constant context switching between browser and AI tool | Stay in your AI tool; query clipboard from within the chat |
| **Best for** | One-off quick pastes | Project research, drafting from collections, multi-day workflows |

The difference isn't marginal — it's a workflow model change. Manual copy-paste treats each AI interaction as an isolated event. Paste MCP treats your entire clipboard history as an ongoing data layer your AI can reference at any time.

## Step-by-Step: Set Up Paste MCP and Build Your First AI Clipboard Workflow

You can go from zero to your first AI clipboard query in under five minutes. Here's exactly how.

### Step 1: Install Paste (if you haven't already)

Download Paste from the Mac App Store. The app offers a free trial — install it, grant the required accessibility permissions (Paste needs clipboard monitoring to build its history), and let it run in your menu bar. Copy a few items — links, notes, code snippets — so test data exists in your history.

[AFFILIATE: Paste] — affiliate program pays 25% on annual subscriptions.

### Step 2: Open MCP Settings in Paste

Click the Paste icon in your menu bar, open Preferences, and navigate to the **MCP** section. If you're on Paste 6.6 or later, this tab appears under "Integrations." You'll see a toggle labeled "Enable MCP."

### Step 3: Enable MCP and Connect Your AI Tool

Flip the toggle to **On**. Paste will present a list of supported AI tools — Claude Desktop, Codex, Cursor, and a custom option for any other MCP-compatible tool. Select yours.

For **Claude Desktop**: Paste auto-generates the `claude_desktop_config.json` entry. Claude will detect Paste as an available MCP resource on next launch.

For **Codex**: Copy the MCP server endpoint Paste shows you and add it to your Codex MCP configuration file. The exact endpoint follows the pattern `mcp-paste://local`.

For **Cursor**: Go to Cursor Settings → MCP Servers → Add. Paste provides the command string to register the local server.

### Step 4: Verify the Connection

Open your AI tool and look for Paste in its available resources. In Claude Desktop, you should see "Paste" listed under connected MCP servers. In Codex or Cursor, run a simple test prompt:

> *"Check Paste for what I copied today. Give me a brief recap of my work."*

If Paste responds with items from your clipboard history, the connection works.

### Step 5: Run Your First Real Workflow

Now put it to work. Try this:

1. Spend 15 minutes researching a topic — copy links, paste quotes, grab screenshots. Don't organize anything yet.
2. Open your AI tool and prompt:

> *"Find all items related to [your topic] in Paste from today. Draft a team update based on them."*

3. The AI reads your clipboard history, identifies relevant items, and produces a draft. You didn't switch windows once to feed it context.

This is the workflow that changes the game. Your AI tool becomes a post-processor for whatever you've already collected — not a blank slate you have to prime.

### Step 6: Configure Exclusions for Sensitive Apps (Important)

Before you go all-in, open Paste's **Excluded Apps** list under Preferences. Add your password manager, banking apps, and any other applications that handle sensitive data. Paste respects these exclusions at the system level — items copied from excluded apps never enter your clipboard history, which means they're never reachable by your AI tool.

## Best Use Cases for Paste MCP

**Best for:**

- **Project research and synthesis** — Collect links, quotes, and notes across days; let AI turn them into a briefing, outline, or draft in one prompt
- **Multi-day writing projects** — Your clipboard naturally accumulates sources, references, and partial drafts. Paste MCP lets AI pick up where you left off without you re-explaining the context
- **Developer debugging sessions** — Stack Overflow snippets, error messages, log output, terminal commands — all in your clipboard history. Ask AI: "Find the three code snippets I copied today and explain how they fit together"
- **Content marketers and writers** — Save headlines, data points, competitor copy, and SEO keywords throughout the week; generate a post outline from everything at once
- **Meeting prep** — Copy links, agenda items, and notes from the days before a meeting; prompt AI to "prepare a summary of what I've been gathering for the Wednesday standup"

**Where it falls short:**

- **iPhone/iPad only** — MCP runs on Mac only. Your mobile clipboard syncs over iCloud, but the MCP server is unavailable on iOS/iPadOS
- **Real-time collaboration** — Paste MCP is single-user. If you share a clipboard pinboard with a teammate, their clips aren't accessible to your AI tool
- **Heavy image workloads** — While Paste stores screenshots, AI tools vary in how well they consume image-based clipboard content via MCP. Text-heavy workflows are where it shines most
- **Learning curve on "open-ended" prompting** — The new capability (ask AI to *find* things in your clipboard) requires a different prompting muscle. Users who stick to "paste this, analyze that" won't get the full value

## Pricing

| Plan | Price | Details |
|---|---|---|
| **Personal (Annual)** | $29.99/year ($2.49/month) | Full access on all devices, includes MCP |
| **Personal (Lifetime)** | One-time purchase | Same features, no recurring charge |
| **Setapp (Bundle)** | From $8.99/month | Paste + 240+ other Mac/iOS apps |
| **Teams** | Contact sales | Shared pinboards, team management |

All plans include the free trial — download from the Mac App Store, use Paste fully for a trial period, then pay only if you want to continue. The MCP feature is included in the Personal plan at no extra cost.

[AFFILIATE: Paste] — the annual plan at $2.49/month is the best value for individual users. Teams pricing is custom; contact Paste sales for quotes.

## FAQ

**Q: Does Paste MCP send my clipboard data to the cloud?**
No. The MCP server runs entirely on your local Mac. Your clipboard history stays on your device or in your private iCloud. No data is transmitted to Paste's servers when you use MCP.

**Q: Can I use Paste MCP with ChatGPT or Gemini?**
Currently, Paste MCP works with tools that support the Model Context Protocol: Claude Desktop, Codex, and Cursor. ChatGPT and Gemini don't support MCP natively yet. If your tool supports custom MCP servers, you can configure it manually via the "other tools" option in Paste's MCP settings.

**Q: What happens if I copy a password while Paste MCP is enabled?**
Nothing — if you've configured it correctly. Add your password manager and any sensitive apps to Paste's **Excluded Apps** list. Items copied from excluded apps never enter your clipboard history. Even without exclusions, Paste runs locally and you control which AI tools have access, and you can revoke that access at any time.

**Q: Does Paste MCP slow down my AI tool?**
No. The MCP query is lightweight — typically a single API call from your AI tool to Paste's local server. Response time is measured in milliseconds. Your AI tool only reads from Paste when you explicitly prompt it to (or when a tool-calling workflow requests clipboard context).

**Q: Can I use Paste MCP for free?**
Paste offers a free trial you can use to test MCP without paying. After the trial, the annual plan at $29.99/year is the entry point. There's no permanent free tier for the full app, but the trial gives you enough time to decide if the MCP workflow fits your needs.

## Conclusion: Turn Your Clipboard Into a Readymade AI Memory Layer

Your clipboard does more work in a day than most knowledge management tools do in a week. With Paste MCP, that work becomes accessible to your AI tools — not as individual items you manually feed, but as a searchable, queryable history your AI can pull from on demand.

The setup takes five minutes. The workflow change is immediate. Stop treating each AI conversation as a blank slate — let your clipboard do the remembering for you.

If you're already using Claude, Codex, or Cursor, install Paste and enable MCP today. Copy through your day normally, then open your AI tool and ask it to find something you copied. The shift from "what do I paste next" to "my clipboard already has the answers" is the kind of productivity leap that feels obvious only after you've tried it.

[LINK: MCP for non-developers — what business users need to know]
[LINK: How to reduce your AI tool stack with the 80/20 rule]
