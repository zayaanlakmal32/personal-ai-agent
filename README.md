# Personal AI Agent

A private AI assistant on Telegram that manages my email, calendar, reminders and job search. It runs entirely on free plans.

## What it does

- **Chat with memory:** remembers facts, tasks and notes between conversations, and automatically picks up lasting facts from chats
- **Voice notes:** English, Sinhala or Singlish, shown as a transcript and then answered
- **Photos and PDFs:** reads receipts, documents and screenshots, and acts on the caption
- **Makes PDFs:** "send me a PDF of…" produces a formatted PDF, sent in the chat (Cloudflare Browser Run, Sinhala supported)
- **Action receipts:** every real action adds a code-written ✅/⏰ line, so the AI can't fake it
- **Gmail:** analyses up to 200 emails at once, reads, drafts, and sends only after I tap Confirm
- **Google Calendar:** checks my schedule, and creates or deletes events only after I tap Confirm
- **Reminders:** one-off, daily or weekly, delivered on time
- **Daily brief:** every morning, covering today's events, the emails that need attention, tasks and reminders
- **Web search:** current answers with sources
- **Job hunting:** reads job alert emails, pulls out each listing, scores how well it fits, tracks applications, sets follow-up reminders, and sends a weekly digest

## How it works

```
Telegram → Cloudflare Worker (webhook) → Queue → Agent
                                                  ├─ Gemini (AI + tool calling)
                                                  ├─ Gmail + Calendar APIs (OAuth)
                                                  ├─ Tavily (web search)
                                                  └─ D1 database (memory, tasks, jobs, reminders)
Cron (every minute) → reminders, daily brief, weekly job digest
```

- **Cloudflare Workers:** hosting, with a cron trigger every minute
- **Cloudflare Queues:** long jobs (e.g. analysing 200 emails) finish without timeouts, and replies stay in order
- **Cloudflare D1:** SQLite database. Tables are created automatically.
- **Gemini API:** the AI, using function calling for tools and JSON mode for job extraction
- **Gmail batch API:** fetches up to 40 emails per request, to stay within the free plan's limits

## Safety

- Only my Telegram user ID can use the bot.
- Telegram webhook calls are checked with a secret token.
- Sending email and creating or deleting events need a Confirm button tap. Pending actions expire after 24 hours and can't run twice.
- Email content is treated as untrusted, so the AI is told never to follow instructions found inside emails.
- No keys are stored in the code. Everything is in Cloudflare secrets.

## Setup

### Secrets (Cloudflare → Worker → Settings → Variables and Secrets)

| Name | What |
|---|---|
| `TELEGRAM_TOKEN` | From @BotFather |
| `ALLOWED_USER_ID` | Your Telegram user ID (from @userinfobot) |
| `WEBHOOK_SECRET` | Any random string (letters, numbers, `_`, `-`) |
| `GEMINI_API_KEY` | From aistudio.google.com |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google Cloud OAuth client (Web application) |
| `TAVILY_API_KEY` | From tavily.com (free plan) |
| `CF_ACCOUNT_ID` / `CF_API_TOKEN` | For PDFs: Cloudflare account ID, plus an API token with "Browser Run - Edit" permission |
| `GEMINI_MODEL` | Optional, defaults to `gemini-flash-latest` |

### Bindings

- **D1 database:** variable `DB`
- **Queue producer:** variable `JOBS` (queue `agent-jobs`)
- **Queue consumer:** this Worker, with batch size 1, retries 0 and max concurrency 1
- **Cron trigger:** `* * * * *`

### Google Cloud

1. Enable the Gmail API and the Google Calendar API.
2. Set up the OAuth consent screen (External) and publish the app. Unpublished apps log out every 7 days.
3. Create an OAuth client (Web application) with this redirect URI: `https://<your-worker>.workers.dev/oauth/callback`

### Connect

1. Deploy `src/index.js`.
2. Open `https://<your-worker>.workers.dev/setup?key=<WEBHOOK_SECRET>` to register the Telegram webhook.
3. Send `/connect_google` to the bot and sign in.

## Commands

| Command | |
|---|---|
| `/tasks` `/notes` `/memory` `/forget` | Memory and to-dos |
| `/reminders` `/brief` | Reminders and daily brief |
| `/jobs` `/applications` `/scan_jobs` | Job hunting |
| `/connect_google` `/disconnect_google` | Google account |

Anything else is plain chat, for example "any important emails today?", "remind me at 9pm to call Amma", or "I applied to job 3".

## Build log

1. Telegram + Gemini chat
2. Memory, tasks and notes (D1)
3. Gmail + Calendar with confirm buttons, batch email fetching, and follow-up context
4. Queue for long jobs, plus reminders and a daily brief (cron)
5. Web search (Tavily), plus job scanning, scoring, tracking and weekly digest
6. Voice notes, photo and PDF reading, automatic long-term memory, PDF creation, and action receipts
