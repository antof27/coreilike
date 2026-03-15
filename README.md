# CoreILike

CoreILike watches [CoreRadio](https://coreradio.online) for new metal releases and sends you a Telegram message only when something matches your taste, based on a pool of favorite bands and genres that you can update anytime by just texting the bot.
This is a personal, non-profit project, built for fun!
---

## How it works

Every hour it scrapes CoreRadio, deduplicates against everything it has already seen, and sends the new releases through an AI filter (Groq / Llama 3.3 70b). If something matches your taste profile, you get a Telegram notification with a short description of why you'd like it. That's it.

The first time it runs it scrapes the full history (~750 releases) so you start with a clean slate. From that point on it only checks what's new.

You can always update your taste profile by messaging the bot directly:

```
add band Spiritbox
remove genre Nu Metal
I don't like Rap Metal anymore
show my favorites
```

It understands plain English and updates immediately.

---

## Architecture

Built entirely in [n8n](https://n8n.io). Two branches in a single workflow sharing the same persistent store:

```
── SCRAPER (every hour) ──────────────────────────────────────────────────────

Schedule Trigger
  → Decide Pages To Fetch       # 50 pages on first run, 1 page after
  → Fetch Coreradio Page        # HTTP POST to coreradio.online
  → Parse HTML Releases         # regex parser, no external dependencies
  → Deduplicate Releases        # persists seen URLs in workflow static data
  → Aggregate New Releases      # collects all pages before the AI call
  → Has New Releases?           # stops here if nothing new
  → Batch Releases For Groq     # loads taste profile from static data
  → Basic LLM Chain (Groq)      # one AI call for the whole batch
  → Split Results               # parses response, one item per release
  → Send Favorite Alert         # Telegram message for each match

── BOT MANAGER (always listening) ───────────────────────────────────────────

Telegram Trigger
  → Parse Intent (Groq)         # understands natural language commands
  → Update Favorites            # reads/writes taste profile in static data
  → Send Reply                  # confirms the update to the user
```

Both branches read and write the same `$getWorkflowStaticData('global')` store, so a taste update via the bot is picked up on the very next scraper run.

---

## Stack

| Component | Tool |
|---|---|
| Workflow automation | [n8n](https://n8n.io) self-hosted |
| AI filtering & NLP | [Groq](https://groq.com) — Llama 3.3 70b |
| Notifications | Telegram Bot API |
| Source | [CoreRadio](https://coreradio.online) |
| Webhook tunnel (dev) | [ngrok](https://ngrok.com) |

---

## Setup

### What you need
- n8n self-hosted instance with a public HTTPS URL
- Groq API key — free at [console.groq.com](https://console.groq.com)
- Telegram bot token from [@BotFather](https://t.me/BotFather)

### Environment variables

```env
TELEGRAM_CREDENTIAL_ID=your_n8n_telegram_credential_id
TELEGRAM_CREDENTIAL_NAME=Telegram Bot

GROQ_CREDENTIAL_ID=your_n8n_groq_credential_id
GROQ_CREDENTIAL_NAME=Groq API

WEBHOOK_URL=https://your-public-url/
```

### Steps

1. Import `CoreILike_merged.json` into n8n
2. Set up Telegram and Groq credentials in n8n
3. Set `WEBHOOK_URL` to your public HTTPS URL and restart n8n
4. Get your Telegram chat ID from [@userinfobot](https://t.me/userinfobot) and hardcode it in the **Send Favorite Alert** and **Send Reply** nodes
5. Activate the workflow

> **Note on the webhook URL:** the Telegram Trigger needs a public HTTPS endpoint. For local dev ngrok works, but free ngrok URLs change on every restart. 

---

## Default taste profile

Ships with my personal defaults, change them anytime via the bot:

**Favorite bands:** Vildhjarta, Allt, Imminence, Karmanjaka, Humanity's Last Breath, Invent Animate, Currents, Lorna Shore, Spiritbox, Sleep Token, Bring Me The Horizon

**Favorite genres:** Djent, Progressive Metalcore, Thall, Progressive Metal, Deathcore, Post-Hardcore

**Not interested in:** Acoustic, Classic Rock, Pop Metal, Nu Metal, Rap Metal

---

## License

MIT — do whatever you want with it.
