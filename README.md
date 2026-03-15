# CoreILike 🎸

An automated metal music curator that monitors [CoreRadio](https://coreradio.online) for new releases, filters them using AI based on your personal taste, and delivers recommendations directly to your Telegram.

---

## What it does

- **Scrapes CoreRadio** every hour for new metal/hardcore releases
- **Deduplicates** releases using persistent state — never notifies you twice about the same release
- **Filters with AI** (Groq / Llama 3.3 70b) — only sends releases that match your taste profile
- **Telegram bot** — talk to it naturally to update your favorite bands and genres on the fly

---

## Architecture

The workflow is built entirely in [n8n](https://n8n.io) and consists of two independent branches in a single workflow:

```
── SCRAPER BRANCH (runs every hour) ─────────────────────────────────────────

Schedule Trigger
  → Decide Pages To Fetch       # first run: 50 pages, subsequent: 1 page
  → Fetch Coreradio Page        # HTTP POST to coreradio.online
  → Parse HTML Releases         # regex-based HTML parser (no external deps)
  → Deduplicate Releases        # persists seen URLs in workflow static data
  → Aggregate New Releases      # collects all pages before AI call
  → Has New Releases?           # stops flow if nothing new
  → Batch Releases For Gemini   # loads taste profile from static data
  → Basic LLM Chain (Groq)      # single AI call for the full batch
  → Split Gemini Results        # parses JSON response, emits one item/release
  → Send Favorite Alert         # Telegram message per relevant release

── BOT MANAGER BRANCH (always listening) ────────────────────────────────────

Telegram Trigger
  → Parse Intent (Groq)         # NLP: understands natural language commands
  → Update Favorites            # reads/writes taste profile in static data
  → Send Reply                  # confirms action to user
```

Both branches share the same `$getWorkflowStaticData('global')` store, so taste profile updates made via the bot are immediately used by the next scraper run.

---

## Features

### Smart first-run scraping
On the first execution, the workflow scrapes up to 50 pages (~750 releases) to build a full history. From then on, it only checks the latest page each hour.

### Single AI call per run
All new releases are batched into one prompt, keeping API usage minimal. Works well within Groq's free tier limits.

### Natural language taste management
Talk to the bot in plain English:

```
add band Spiritbox
remove genre Nu Metal
I don't like Rap Metal anymore
add Lorna Shore to my favorites
show my favorites
```

The bot understands intent and updates your profile immediately.

### Persistent deduplication
Seen release URLs are stored in n8n's workflow static data. The scraper never processes the same release twice across runs.

---

## Stack

| Component | Tool |
|---|---|
| Workflow automation | [n8n](https://n8n.io) (self-hosted) |
| AI filtering & NLP | [Groq](https://groq.com) — Llama 3.3 70b |
| Notifications | Telegram Bot API |
| Scraping target | [CoreRadio](https://coreradio.online) |
| Webhook tunnel (dev) | [ngrok](https://ngrok.com) |

---

## Setup

### Prerequisites
- n8n self-hosted instance
- Groq API key (free at [console.groq.com](https://console.groq.com))
- Telegram bot token (from [@BotFather](https://t.me/BotFather))
- Public HTTPS URL for n8n (required for Telegram webhook)

### Environment variables

```env
# Telegram
TELEGRAM_CREDENTIAL_ID=your_n8n_telegram_credential_id
TELEGRAM_CREDENTIAL_NAME=Telegram Bot

# Groq
GROQ_CREDENTIAL_ID=your_n8n_groq_credential_id
GROQ_CREDENTIAL_NAME=Groq API
```

### Installation

1. Clone this repo
2. Import `CoreILike_merged.json` into your n8n instance
3. Configure credentials in n8n for Telegram and Groq
4. Set `WEBHOOK_URL` in your n8n environment to your public HTTPS URL
5. Hardcode your Telegram chat ID in the **Send Favorite Alert** and **Send Reply** nodes (get it from [@userinfobot](https://t.me/userinfobot))
6. Activate the workflow

### Getting your Telegram chat ID
Send any message to [@userinfobot](https://t.me/userinfobot) on Telegram — it replies instantly with your chat ID.

---

## Default taste profile

The workflow ships with a default profile that you can override at any time via the bot:

**Favorite bands:** Vildhjarta, Allt, Imminence, Karmanjaka, Humanity's Last Breath, Invent Animate, Currents, Lorna Shore, Spiritbox, Sleep Token, Bring Me The Horizon

**Favorite genres:** Djent, Progressive Metalcore, Thall, Progressive Metal, Deathcore, Post-Hardcore

**Not interested in:** Acoustic, Classic Rock, Pop Metal, Nu Metal, Rap Metal

---

## Known limitations

- The Telegram Trigger requires a public HTTPS URL. On local dev, use ngrok (note: free ngrok URLs change on restart — use a static domain or VPS for permanent deployment)
- CoreRadio's HTML structure may change over time, requiring updates to the parser regex in `Parse HTML Releases`
- Groq's free tier has rate limits — if you scrape many pages at once, the single-batch approach keeps usage well within limits

---

## License

MIT