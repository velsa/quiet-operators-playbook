# WhatsApp Read-Only Monitor with Telegram Inbox

**Intent:** Connect WhatsApp to your agent as a read-only channel. All incoming messages are forwarded to Telegram for review. You decide what to reply — the agent never sends on WhatsApp without explicit approval.

**Scope:** 1-2 hour setup. Solves the "agent replies to your contacts on your behalf" problem.

**What you get:**
- Full visibility into incoming WhatsApp messages via Telegram
- Contacts and groups tracked with priority levels
- Inbox command: shows pending conversations grouped by status
- Agent drafts replies on request, sends only when you confirm

---

## The problem

Most WhatsApp + AI agent setups fail the same way: the agent treats incoming messages as input and generates replies automatically. This means your contacts get AI responses they did not ask for, from what looks like your personal account.

WhatsApp is personal. Auto-replies break trust.

---

## Architecture

```
WhatsApp (read) → OpenClaw gateway → Agent (arminer)
                                          ↓
                                    Telegram (you)
                                          ↓
                                    "Reply X to Y"
                                          ↓
                                    Agent drafts → you approve
                                          ↓
                                    WhatsApp (write, only on approval)
```

---

## Setup

### 1. Connect WhatsApp to OpenClaw

```bash
openclaw channels login --channel whatsapp
# Scan QR code with your phone
```

### 2. Set WhatsApp as read-only in your agent config

Add to `SOUL.md` (your agent's identity file):

```
## WhatsApp Rules (MANDATORY)

WhatsApp is READ-ONLY. Never send anything on WhatsApp without explicit approval.

When a message arrives:
1. Never reply on WhatsApp
2. Forward summary to Telegram: "📱 WhatsApp from [name]: [summary]"
3. Wait for instruction: reply / hold / ignore
4. Draft replies on request, send only after confirmation
```

### 3. Build a contacts file

Create `whatsapp-contacts.md`:

```markdown
## Contacts
- Nadya (sister) — HIGH priority, flag immediately
- Boss — Normal, flag when needs reply

## Groups  
- Work group — Flag if someone asks me directly

## Channels
- News channel — Summarize, extract event dates
```

### 4. Inbox command

Tell your agent: whenever you say "inbox", show:
- 🔴 Needs reply
- 🟡 On hold
- ⚫ Ignore until next message
- ✅ Done

---

## Limitations

**Read receipts:** WhatsApp marks messages as read when the gateway processes them. There is no way to suppress this via the WhatsApp Web API. Workaround: use Telegram as your primary inbox instead of watching WhatsApp unread counts.

**Contact names:** OpenClaw does not sync your contacts list. You will see phone numbers, not names, until you build your contacts file manually. Ask your agent "who is +972XXXXXXX?" once, it saves the answer.

**Groups:** The agent sees group messages but has no group history before it connected. Context builds over time.

---

## What works well

- Zero accidental replies
- Full audit trail (agent logs every message it saw)
- Priority routing (sister's messages get flagged immediately)
- Works headless, no browser, no phone in hand

Full playbook: https://github.com/velsa/quiet-operators-playbook
