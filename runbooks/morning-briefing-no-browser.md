# Morning Briefing Without a Browser

**Intent:** Deliver a daily morning summary (email, calendar, notifications) to Telegram at a fixed time — no browser, no GUI, runs headless on a remote server.

**Scope:** 30-60 minute setup. Saves 15-20 minutes of morning context-switching every weekday.

**What you get:** 3 Telegram messages at 9am:
1. Unread email summary — sender, subject, anything urgent flagged
2. Today's calendar events (skipped entirely if nothing scheduled)
3. Moltbook / community notifications

---

## The problem with browser-based approaches

Most morning briefing setups use browser automation to scrape Gmail. This breaks in headless server environments, requires a display, and is fragile to UI changes. API-first is more reliable and runs anywhere.

---

## Prerequisites

- Linux server with cron and curl
- [gog](https://github.com/bitful-pannul/gog) — Gmail and Google Calendar CLI
- Google OAuth credentials (one-time setup via `gog auth`)
- Moltbook API key (optional)

---

## Step 1: Set up gog

```bash
npm install -g @bitful/gog
gog auth --account yourname@gmail.com

# Test
gog gmail search "is:unread newer_than:1d" --account yourname@gmail.com --json
```

**Headless gotcha:** gog stores OAuth tokens in a keyring. In cron environments, set:

```bash
export GOG_KEYRING_PASSWORD=your_password
```

Add to ~/.zshrc. Without this, gog hangs waiting for a TTY prompt.

---

## Step 2: The script

```bash
#!/bin/bash
source ~/.zshrc

EMAIL_ACCOUNT="yourname@gmail.com"

# EMAIL
EMAILS=$(gog gmail search "is:unread newer_than:1d" \
  --account "$EMAIL_ACCOUNT" --json --results-only 2>/dev/null)
EMAIL_COUNT=$(echo "$EMAILS" | jq 'length')

if [ "$EMAIL_COUNT" -gt 0 ]; then
  echo "📧 $EMAIL_COUNT unread:"
  echo "$EMAILS" | jq -r '.[] | "• \(.from | split("<")[0] | rtrimstr(" ")) — \(.subject)"' | head -10
fi

# CALENDAR (skip entirely if no events)
EVENTS=$(gog calendar events --account "$EMAIL_ACCOUNT" --today --json --results-only 2>/dev/null)
EVENT_COUNT=$(echo "$EVENTS" | jq 'length')

if [ "$EVENT_COUNT" -gt 0 ]; then
  echo "📅 Today:"
  echo "$EVENTS" | jq -r '.[] | "• \(.start.dateTime // .start.date | .[11:16]) — \(.summary)"'
fi

# MOLTBOOK
if [ -n "$MOLTBOOK_API_KEY" ]; then
  NOTIF_COUNT=$(curl -s "https://www.moltbook.com/api/v1/notifications" \
    -H "Authorization: Bearer $MOLTBOOK_API_KEY" | jq '.notifications | length')
  [ "$NOTIF_COUNT" -gt 0 ] && echo "🦞 $NOTIF_COUNT Moltbook notifications"
fi
```

---

## Step 3: Schedule it

```bash
crontab -e
# 9am Israel time (UTC+2), Sunday-Thursday
0 7 * * 0-4 /bin/bash /path/to/morning-update.sh
```

---

## What makes this reliable

- No browser — runs on any headless Linux box
- Calendar skip logic — no events, no message, no noise
- Source your shell profile in the script — cron does not do it for you
- JSON-first — all parsing via jq, not screen-scraping

---

## Troubleshooting

**gog hangs in cron:** Missing GOG_KEYRING_PASSWORD. Source ~/.zshrc explicitly in script.

**Empty results:** Token expired. Run `gog auth` interactively to refresh.

**Wrong time:** Cron runs UTC. Israel is UTC+2 (winter) or UTC+3 (summer).

Full playbook: https://github.com/velsa/quiet-operators-playbook
