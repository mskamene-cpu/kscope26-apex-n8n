# RMA Copilot — A WhatsApp Returns Bot with Oracle APEX, n8n & AI

A working "Returns Copilot": a customer sends a WhatsApp message — *"I need to return ORD-10452, it arrived cracked"* — and a few seconds later gets a reply with an RMA number. Behind that one message, Twilio catches it, n8n orchestrates it, OpenAI reads the intent, and **Oracle APEX makes every business decision and writes every record**.

> **AI classifies. n8n orchestrates. APEX decides.**

This repo is the companion code to the Kscope session *When APEX Talks Back* and the step-by-step blog post. The blog is the full tutorial; this README is the map.

📖 **Full walkthrough:** [When an APEX App Talks Back — build it yourself](https://mskamene.hashnode.dev/whatsapp-apex-n8n-ai)

---

## Architecture

```
WhatsApp ──► Twilio ──► n8n ──► Oracle APEX (ORDS REST)
                         │            │
                         │            ├─ validates the order
                         │            ├─ checks return eligibility   ◄── decisions live here
                         │            ├─ creates / approves the RMA
                         │            └─ owns conversation state + audit trail
                         └─ OpenAI (gpt-4o-mini): intent + entity extraction only
```

Each tool stays in its lane. The AI never decides whether a return is allowed — it guesses what the customer wants, and APEX, reading `APP_CONFIG` and the order data, decides. Conversation state lives in APEX, so the n8n workflow is completely stateless.

---

## What's in here

```
.
├── apex/                 # APEX application export — install via App Builder → Import (incl. Supporting Objects)
├── n8n/
│   ├── RMA_Copilot_v2.json   # the workflow the blog walks through (simplified, ~22 nodes)
│   └── RMA_Copilot.json              # full production flow (multi-turn, approval split, per-branch state)
└── README.md
```

**Two workflow versions.** Start with `RMA_Copilot_v2_rebuild.json` — it's the legible version the blog teaches end to end. Graduate to `RMA_Copilot.json` once that's boring; it adds multi-turn order-number collection, the auto-approve / manager-approval split, and per-branch state resets.

---

## Prerequisites

All four have a free tier good enough to build this:

- **Oracle APEX / Autonomous Database** — your own instance, or a free workspace at [apex.oracle.com](https://apex.oracle.com)
- **n8n** — [cloud](https://n8n.io) or self-hosted
- **Twilio** — free trial; we use the WhatsApp **Sandbox** (no Meta approval needed)
- **OpenAI** — an API key with a little credit (uses `gpt-4o-mini`, costs ~nothing)

---

## Quick start

The blog has the click-by-click version. The short form:

1. **Install the APEX app.** App Builder → Import → choose the export in `apex/` → **install Supporting Objects** (this creates the tables, seed data, and config).
2. **Publish the ORDS endpoints** (done for you by Supporting Objects; see the table below).
3. **Create an OAuth client** in your schema (`OAUTH.create_client`, `client_credentials`) for n8n to authenticate with.
4. **Configure `APP_CONFIG`** and add the Twilio **Web Credential** (see below).
5. **Twilio sandbox** — join the sandbox, grab your Account SID + Auth Token.
6. **n8n** — import a workflow from `n8n/`, create the three credentials, point the HTTP nodes at your ORDS base URL, then **save + activate**.
7. **Wire Twilio → n8n** — set the sandbox's "When a message comes in" to your n8n production webhook URL (POST).
8. **Message the sandbox** and watch the RMA appear in APEX.

---

## Configuration

### `APP_CONFIG` 

| CONFIG_KEY | What it controls |
|---|---|
| `TWILIO_FROM_WHATSAPP` | WhatsApp sender number (the sandbox number, with `+`) |
| `APPROVAL_THRESHOLD_CAD` | Order total above which a return needs manager approval (e.g. `500`) |

---

## ORDS endpoints

All under your ORDS base, e.g. `https://YOUR-ADB.adb.YOUR-REGION.oraclecloudapps.com/ords/your_schema`:

| Method | Path | Purpose |
|---|---|---|
| `POST`  | `/api/conversations/inbound-message` | Log inbound message, return conversation + state |
| `POST`  | `/api/conversations/outbound-message` | Log outbound message, return conversation + state |
| `GET`   | `/api/orders/{id}/return-eligibility` | Eligibility decision |
| `POST`  | `/api/returns` | Create the RMA |
| `POST`  | `/api/returns/{id}/approve` | Approve a pending RMA *(production flow)* |
| `POST`  | `/api/conversations/outbound-message` | Log the reply sent |
| `PATCH` | `/api/conversations/{id}/state` | Advance conversation state |
| `POST`  | `/api/events/audit` | Append an audit event |
| `GET`  | `/api/orders/:order_number` | Looks up an order by order number |


---

## ⚠️ Before you commit or publish

- The workflow JSONs here use placeholders (`YOUR_SERVER`, `YOUR_WORKSPACE`, `YOUR_TWILIO_AC`). **Swap in your own values locally — don't commit real ones.**
- n8n credential *secrets* are not included in a workflow export, but **hardcoded values, your Account SID in the Twilio URL, and pinned test data are.** Clear pinned data and scan the JSON for `token`, `Bearer`, `apiKey`, and phone numbers before pushing.

---

## Links

- 📖 Blog: [When an APEX App Talks Back](https://github.com/mskamene-cpu/kscope26-apex-n8n)


*Built with Oracle APEX, n8n, Twilio, and OpenAI.*
