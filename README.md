#  Escalator — AI Escalation Routing for Support Teams

> An n8n-powered AI agent that reads support conversations, writes production-grade escalation cases, and routes them to the right team's Slack channel — so customers never have to repeat themselves.

Built by **Tobi Kehinde**,  7 years in customer support & success (high-volume call centers, remote U.S. healthcare operations, QA supervision), now bridging frontline ops experience with AI automation.

---

## 🎯 The Problem: Ticket Ping-Pong

In most support orgs, escalation works like this:

1. Tier 1 troubleshoots with the customer (restart, re-login, resend…).
2. It fails. Tier 1 writes: *"Customer says it's not working. Please help."*
3. Tier 2 opens the ticket 4 hours later, reads 50 chat messages, and asks the customer for the transaction ID **again**.
4. The customer repeats their story. Everyone's MTTR bleeds. Everyone's CSAT bleeds.

The handoff — not the incident — is where support teams lose hours and customers.

## ✅ The Solution

**Escalator** sits between Tier 1 and Tier 2. When an agent escalates, it:

- 📥 Receives the ticket + full chat transcript via webhook (simulates Zendesk / Intercom / custom CRM)
- 🧠 Uses an LLM to read the conversation and draft a structured escalation case
- 🔍 Extracts hard facts: transaction IDs, terminal IDs, error codes, timestamps — and honestly reports `not provided` when missing (anti-hallucination by prompt design)
- 📝 Writes a case description that includes **"steps already taken"** and an explicit *"do NOT re-ask the customer for X"* instruction
- 🔀 Routes the case to the correct department Slack channel based on AI-classified queue
- 🚨 Pages the on-call human (Telegram) for P1 fires
- 🛡️ Suppresses duplicate ticket submissions (idempotency by `ticket_id`)
- 📊 Logs everything to a live Google Sheets escalation dashboard

## 🏗️ Architecture

```
[CRM / Agent clicks "Escalate"]
            │  POST webhook: { ticket_id, customer_name, transcript[], agent_notes }
            ▼
      ┌───────────┐
      │  Webhook  │
      └─────┬─────┘
            ▼
   ┌─────────────────┐  duplicate?   ┌──────────────────┐
   │  Dedupe Check   │──────────────▶│  Stop / log only │
   │ (ticket_id vs   │               └──────────────────┘
   │  Sheets lookup) │
   └───────┬─────────┘
           │ new ticket
           ▼
   ┌─────────────────┐
   │  OpenAI (LLM)   │  reads transcript → drafts structured case JSON
   └───────┬─────────┘
           ▼
   ┌─────────────────┐
   │  Parse Case JSON│
   └───────┬─────────
           ├──────────────────────┐
           ▼                      ▼
  ┌─────────────────┐   ┌──────────────────────────────┐
  │  Google Sheets  │   │  Slack Router (queue→channel)│
  │  Escalations DB │   │  #tech-disputes · #billing    │
  │  🔴/🟢 dashboard│   │  #hardware · #app-bug · #ops  │
  └─────────────────┘   │  fallback: #support-general   │
                        └──────────────────────────────┘
```

**Companion module (v1 — customer-facing triage):** form/email intake → AI classifies category + sentiment → live dashboard → angry customers trigger an instant SLA page; calm customers receive an automated acknowledgement email with a ticket reference.

## ✨ Features

| Feature | Detail |
|---|---|
| AI case drafting | Structured JSON: `issue_title`, `root_cause_hypothesis`, `queue`, `priority`, `key_facts`, `steps_already_taken`, `customer_sentiment`, `case_description` |
| Anti-hallucination | Missing facts return `not provided` instead of invented IDs |
| Queue-based Slack routing | Mapping-object routing with safe fallback channel |
| P1 paging | P1 cases page the on-call human on Telegram |
| Idempotency | Duplicate `ticket_id` submissions are suppressed before any LLM call (cost-safe) |
| Live dashboard | Google Sheets with 🔴 ESCALATED / 🟢 Queued statuses |
| Auto-acknowledgement | Calm tickets get an instant human-toned email with reference ID |

## 🧰 Tech Stack

- **n8n** (cloud) — orchestration
- **OpenAI API** — case drafting & classification
- **Slack API** (bot token) — department routing
- **Google Sheets API** — escalation database & dashboard
- **Telegram Bot API** — on-call paging
- **Gmail** — customer acknowledgements
- **Webhooks / JSON** — CRM integration layer

## 📦 Example

**Input (webhook payload):**
```json
{
  "ticket_id": "TKT-88492",
  "customer_name": "Mrs. Adaeze Okafor",
  "channel": "live_chat",
  "transcript": [
    {"from": "customer", "text": "My POS printed nothing but the customer's account was debited N15,500!"},
    {"from": "agent", "text": "Can you confirm the terminal ID on the back of the device?"},
    {"from": "customer", "text": "It's T-9921. The transaction was at 2:04pm, reference 8849201."},
    {"from": "agent", "text": "I restarted the app and re-sent the print job, but it still fails with 'Printer Timeout'."}
  ],
  "agent_notes": "Restart + re-login done. Print job resend failed. Error 504 Printer Timeout on terminal T-9921."
}
```

**Output (routed to `#tech-disputes`):**
```
🚨 P1 ESCALATION — Tech-Disputes
Issue: POS debited account, printer failed, customer without receipt
Facts: Tx `8849201` | Terminal `T-9921` | Err `504 Printer Timeout`
Already tried: Restarted POS application, Re-logged into terminal, Re-sent print job
Sentiment: Frustrated

Customer's POS transaction at 2:04pm on terminal T-9921 debited N15,500 but failed
to print a receipt (error 504 Printer Timeout)… Do NOT ask the customer for
terminal ID, reference, or to restart/re-login again — these actions have been completed.
```

## 🚀 Reproduce It

1. **Slack:** create channels `#tech-disputes`, `#hardware`, `#billing`, `#app-bug`, `#ops`, `#support-general`. Create a Slack app with `chat:write`, `channels:read` scopes; install; invite the bot to each channel; copy the `xoxb-` bot token.
2. **Google Sheets:** create a sheet with an `Escalations` tab, headers: `Timestamp | Ticket ID | Customer | Issue Title | Queue | Priority | Sentiment | Case Description`.
3. **n8n:** import `workflow/case-builder.json` from this repo.
4. **Credentials:** connect OpenAI, Google Sheets, Slack (bot token), Telegram, Gmail.
5. **Prompts:** paste the system prompt from `prompts/case-architect.md`.
6. **Activate** the workflow and copy the production webhook URL.
7. **Test:** `curl -X POST -H "Content-Type: application/json" -d @mock-payloads/pos-dispute.json <YOUR_WEBHOOK_URL>`
8. Watch `#tech-disputes` light up. Fire the billing payload; watch it route to `#billing`. Fire any payload twice; watch the duplicate get suppressed.

## 🧠 Design Decisions (Lessons Learned)

- **Dedupe before the LLM.** Duplicate checks run before any AI call — retries never burn tokens.
- **Email is async, chat is sync.** Instant acknowledgement emails build trust; instant chat replies feel robotic. Pacing is per-channel by design.
- **Bots need identities.** Escalations post as the bot, never as a human user — teams must always know who (or what) is speaking.
- **The fallback channel is the safety net.** Unmapped queues land in `#support-general`, never in the void.
- **"Do NOT re-ask" is the product.** The single highest-value sentence in every case description.

## 🗺️ Roadmap

- [ ] Slack "Acknowledge" button → webhook → sheet status update (human-in-the-loop)
- [ ] SLA breach timer (Wait node + status check → re-page)
- [ ] WhatsApp & Intercom intake adapters
- [ ] RAG-powered FAQ auto-resolution for known issues
- [ ] MTTR & escalation-volume metrics dashboard

## 👤 About the Author

**Tobi Kehinde** — Customer Support & Success Specialist (7+ yrs) · Lagos, Nigeria
Remote U.S. healthcare support · QA supervision · 91–100% QA scores · Now building AI automation for support operations.
📧 iamtobi001@gmail.com · 📱 +234 907 606 6694 · [LinkedIn](https://www.linkedin.com/in/tobi-kehinde-2a11a7213)

##  License

MIT — use it, fork it, ship it. A credit line makes my day.
