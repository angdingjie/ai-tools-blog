---
title: "How to Build a Local-First AI Research Assistant: Whispering + Ollama + Obsidian Step by Step"
date: 2026-06-30T08:00:00+08:00
draft: false
categories: ["Artificial Intelligence"]
tags: ["AI tools", "productivity"]
description: "You take notes in Obsidian. You dictate ideas into your phone. You keep telling yourself you'll process that research pile later. But later never comes, an"
---

You take notes in Obsidian. You dictate ideas into your phone. You keep telling yourself you'll process that research pile later. But later never comes, and you're paying $30/month for Otter.ai and $20/month for Notion AI just to keep your head above water.

There's a better way — and it runs entirely on your machine.

Whispering (YC-backed, 591 upvotes on HN) brings open-source, local-first dictation to your desktop. Pair it with Ollama for a local LLM that summarizes and extracts insights, and wire it into Obsidian so every thought lands in your knowledge base automatically. No data leaves your computer. No subscription fees. No vendor lock-in.

Here's exactly how to build that stack — the tools, the config, and the prompt chain that makes it work.

## What Is a Local-First AI Research Assistant?

A local-first AI research assistant is a pipeline of open-source tools that runs entirely on your hardware. You speak your thoughts, the system transcribes, summarizes, and files them into your note-taking app — all without touching a cloud server.

The core trio:

- **Whispering** — an open-source desktop app that provides real-time, local transcription using OpenAI's Whisper model. Built by the Epicenter team (YC S23), it hit #1 on Hacker News with 591 points and 152 comments. It's a system-level microphone listener: press a hotkey, speak, and it transcribes. No internet required after the initial model download.

- **Ollama** — the easiest way to run local LLMs. Download models like Llama 3.1, Mistral, or Gemma with a single command. It exposes a REST API that other tools can call for summarization, extraction, and rewriting.

- **Obsidian** — the most popular local knowledge base (3M+ users). Everything is plain Markdown files in a local folder. No cloud, no lock-in. Perfect for the local-first ethos.

The chain works like this:

```
Your voice → Whispering (transcription) → Ollama (summarization/extraction) → Obsidian (permanent note storage)
```

Each piece is replaceable. Don't like Ollama? Swap in llama.cpp or LM Studio. Prefer Logseq to Obsidian? The notes are just files — it works with any Markdown-based tool.

## Whispering vs Otter.ai vs Notion AI — Comparison Table

| Feature | Whispering (Open Source) | Otter.ai (Paid) | Notion AI (Paid) |
|---|---|---|---|
| **Price** | Free | $16.99–$30/month | $10/month (add-on) |
| **Runs locally** | ✅ Yes | ❌ Cloud only | ❌ Cloud only |
| **Privacy** | 100% local. No data leaves your machine | Processed on Otter servers | Processed on Notion/OpenAI servers |
| **Internet required** | No (after model download) | Yes | Yes |
| **Real-time transcription** | ✅ Yes | ✅ Yes | ❌ No (text-based AI only) |
| **Custom LLM integration** | ✅ Any local or remote LLM via API | ❌ Locked to Otter's AI | ❌ Locked to Notion's models |
| **Export format** | Plain text, configurable | Otter format, limited export | Notion blocks only |
| **Hotkey trigger** | ✅ Yes | ❌ No (always-listening) | ❌ No |
| **Knowledge base storage** | Your files (Obsidian/any) | Otter cloud | Notion cloud |
| **Open source** | ✅ MIT license | ❌ Proprietary | ❌ Proprietary |

**Winner by privacy and price:** Whispering. Otter.ai is more polished if you need speaker identification and team collaboration, but for a solo researcher who wants privacy and zero subscription fees, Whispering wins hands down.

## Step-by-Step: Set Up Your Local Research Assistant

### Step 1: Install Ollama and Pull a Model

Ollama runs on macOS, Linux, and Windows. Install it from [ollama.com](https://ollama.com), then pull a model suited for summarization:

```bash
# Install (macOS/Linux)
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model — Llama 3.1 8B is fast enough for most hardware
ollama pull llama3.1:8b

# For better summarization, try Qwen2.5 7B (handles 32K context)
ollama pull qwen2.5:7b

# Test it works
ollama run llama3.1:8b "Summarize this in one sentence: Local AI tools keep your data private."
```

You need at least 8GB of RAM for 7B-parameter models. With 16GB+, you can run 13B models comfortably.

### Step 2: Install and Configure Whispering

Clone and set up Whispering:

```bash
git clone https://github.com/epicenter-so/epicenter.git
cd epicenter/apps/whispering
npm install
```

Whispering uses a system tray icon with a hotkey (default: `Cmd+Shift+R` on Mac, `Ctrl+Shift+R` on Windows/Linux). Press it to start recording, press again to stop and transcribe.

After installation, configure the output pipeline. Whispering can send transcribed text to:

1. A **local file** — append every transcription to `~/Research/Inbox.md`
2. The **clipboard** — paste it into any app
3. A **webhook** — POST to your own endpoint for custom processing

For our stack, configure it to write to a file that a script will pick up.

```bash
# Whispering config (config.json)
{
  "hotkey": "CmdOrCtrl+Shift+R",
  "model": "base",  # or "small" or "medium" for better accuracy
  "output": {
    "mode": "file",
    "path": "~/research/inbox/raw-transcriptions.md",
    "append": true
  }
}
```

The `base` model is fast and accurate enough for most environments. Use `small` if you have a GPU. Use `medium` or `large` only if you have significant background noise or technical jargon.

### Step 3: Create the Processing Script

This is the glue that connects Whispering → Ollama → Obsidian. When your inbox file gets a new entry, this script reads it, sends it to Ollama for processing, and writes the result into your Obsidian vault.

Save this as `~/research/process-transcription.py`:

```python
#!/usr/bin/env python3
import json
import urllib.request
from pathlib import Path
from datetime import datetime

INBOX = Path("~/research/inbox/raw-transcriptions.md").expanduser()
VAULT = Path("~/obsidian-vault/Research").expanduser()
OLLAMA_URL = "http://localhost:11434/api/generate"
MODEL = "llama3.1:8b"

def summarize(text):
    prompt = f"""You are a research assistant. Given raw transcription text,
extract key insights in these sections:
- **Summary** (2-3 sentences)
- **Key points** (bullet list, max 5)
- **Action items** (if any)
- **Tags** (3-5 comma-separated keywords)

Raw text: {text}"""
    
    payload = {
        "model": MODEL,
        "prompt": prompt,
        "stream": False,
        "options": {"temperature": 0.3}
    }
    req = urllib.request.Request(
        OLLAMA_URL,
        data=json.dumps(payload).encode(),
        headers={"Content-Type": "application/json"}
    )
    with urllib.request.urlopen(req) as resp:
        result = json.loads(resp.read())
    return result["response"]

def save_to_obsidian(summary, raw_text):
    date = datetime.now().strftime("%Y-%m-%d")
    slug = datetime.now().strftime("%H%M%S")
    filename = VAULT / f"{date}_voice-note-{slug}.md"
    
    content = f"""---
date: {date}
source: voice-dictation
tags: [voice-note, inbox]
---

{summary}

---
## Raw transcription
> {raw_text.strip()}
"""
    VAULT.mkdir(parents=True, exist_ok=True)
    filename.write_text(content)
    print(f"Saved: {filename}")

if __name__ == "__main__":
    raw = INBOX.read_text()
    if not raw.strip():
        print("No new transcriptions.")
        exit(0)
    
    summary = summarize(raw)
    save_to_obsidian(summary, raw)
    # Clear inbox after processing
    INBOX.write_text("")
```

Make it executable and set up a cron job (or use a file watcher like `fswatch` on macOS):

```bash
chmod +x ~/research/process-transcription.py

# macOS — watch with fswatch
brew install fswatch
fswatch -o ~/research/inbox/raw-transcriptions.md | xargs -n1 ~/research/process-transcription.py

# Linux — watch with inotifywait
sudo apt install inotify-tools
while inotifywait -e modify ~/research/inbox/; do
  ~/research/process-transcription.py
done
```

### Step 4: Configure Your Obsidian Vault

Make your vault work with this pipeline:

1. **Create a `Research` folder** — this is where processed notes land.
2. **Install the Dataview plugin** — it queries notes by tags and dates.
3. **Build a dashboard** using Dataview:

```
## 📥 Recent Voice Notes
```dataview
LIST
FROM #voice-note
SORT file.ctime DESC
LIMIT 20
```

4. **Set up a daily review note** that aggregates voice notes from today. Create a template:

```markdown
## Voice Notes — {{date}}

```dataview
TABLE summary AS Summary, file.ctime AS Time
FROM "Research"
WHERE source = "voice-dictation" AND date = "{{date}}"
SORT file.ctime
```

Now every dictation, idea, or research snippet is automatically transcribed, summarized, tagged, and filed into your vault — ready for your daily review.

## Best For / Worst For

### Best For

- **Solo researchers and PhD students** — dictating literature notes, interview quotes, and research ideas without breaking flow.
- **Indie hackers documenting decisions** — speak design rationale while coding, have it auto-summarized into your project journal.
- **Privacy-conscious professionals** — lawyers, doctors, journalists handling sensitive information that can't go through cloud services.
- **Knowledge workers building a second brain** — the pipeline feeds directly into any Zettelkasten or PARA workflow in Obsidian.

### Worst For

- **Team collaboration** — Whispering is a single-user tool. Otter.ai handles speaker identification and team libraries better.
- **Non-technical users** — the setup requires command line comfort and basic scripting. Not a turnkey solution yet.
- **Meeting transcription** — Whispering captures one audio stream. Use it for your own dictation, not team meetings.
- **Users with no GPU/RAM** — 7B models need 8GB RAM minimum. Run on CPU (slower) or use smaller models like Gemma 2B.

## Pricing

| Component | Price | Notes |
|---|---|---|
| Whispering | **Free** (MIT) | Open source, no paid tier |
| Ollama | **Free** (MIT) | Open source, no paid tier |
| Obsidian | **Free** (commercial use license) | Optional Sync: $5/month, Publish: $10/month |
| LLM Model | **Free** (open-weight) | Llama 3.1, Qwen2.5, Mistral — all free to download |
| **Total** | **$0–$5/month** | Obsidian Sync optional; everything else is free |
| vs. Otter.ai + Notion AI | **$37–$50/month** | Cloud subscriptions |

You save $35–$50/month going local. Over a year, that's $420–$600 — plus the peace of mind that your research data never touches a third-party server.

## FAQ

### Do I need a GPU to run this stack?
No. Whispering runs with CPU-based Whisper (a few seconds delay for transcription). Ollama runs 7B models at 5-15 tokens/second on CPU with 16GB RAM. A GPU makes everything faster but isn't required.

### Can I use Whispering with other note apps?
Yes. The pipeline writes to plain Markdown files. Use it with Logseq, Dendron, Foam, or even Notion (via their API). You just need to point the output file to the right folder.

### How accurate is local transcription vs. cloud services?
[LINK: Whisper vs Otter.ai accuracy comparison]. OpenAI's Whisper Large-v3 is within 1-2% WER of cloud services like Otter.ai. The `small` and `medium` models trade a few points of accuracy for speed but remain usable for note-taking.

### Can I use a different local LLM with this setup?
Absolutely. Swap the model in the Python script. Try Mistral 7B for faster inference, Qwen2.5 7B for better context handling, or DeepSeek Coder if your research involves code. All available via `ollama pull`.

### Does Whispering work on Windows?
Yes. Whispering is built with Electron and runs on macOS, Windows, and Linux. The hotkey defaults differ per platform (`Ctrl+Shift+R` on Windows/Linux, `Cmd+Shift+R` on macOS).

## Conclusion

You don't need to pay $50/month and trust your research data to cloud servers just to take voice notes. With Whispering, Ollama, and Obsidian, you get a local-first research pipeline that transcribes, summarizes, and organizes your thoughts — all on your own hardware.

The setup takes about an hour. The savings run forever.

Start with the basics: install Ollama, pull a model, and get Whispering running. Add the processing script when you're comfortable. [LINK: Advanced AI workflow automation guide] Extend it with [AFFILIATE: Focusmo] for pomodoro-based processing triggers, or [AFFILIATE: TabTabTab] if you want the same AI-powered approach for your spreadsheets.

**Next step:** Clone Whispering, run `npm install`, and dictate your first research note today. Your future self — and your privacy — will thank you.
