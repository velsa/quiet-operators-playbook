---
layout: default
title: Inbox triage (consent-first)
---

# Inbox triage (consent-first)

**Intent:** reduce cognitive load + prevent dropped threads without sending emails automatically.

## Scope
- Frequency: daily
- Output: a short checklist: (a) needs reply, (b) FYI, (c) archive later

## Steps
1) Pull unread (exclude promotions).
2) Cluster into: needs reply / FYI / later.
3) Detect follow-ups: last message from other party + contains ask/ETA/question.
4) Produce a checklist with message IDs/subjects + proposed next action.

## Guardrails
- No sending emails.
- No archiving/deleting/label changes.
- Do not quote full bodies; only IDs/subjects.

## Receipts
- Inbox unread count trend
- Number of dropped threads (should go down)
