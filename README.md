# n8n Workflows for Accounting & Bookkeeping Firms

Backend automation workflows built on [n8n](https://n8n.io) for Australian accounting and bookkeeping firms. Open-source, self-hostable, designed for firms running Xero and MYOB.

Built by [Gimhana Wijethunga](https://gimhana.dev) — CS student at Swinburne University, Melbourne.

---

## What's in this repo

| # | Workflow | What it does | Trigger |
|---|---------|-------------|---------|
| 01 | [Client Document Collection](#01--client-document-collection) | Sends document requests to clients with escalating reminders at Day 3, 7, and 14 | Webhook |
| 02 | [Invoice OCR → Xero/MYOB](#02--invoice-ocr--xero--myob) | Watches inbox for supplier invoices, extracts data via Mindee OCR, creates draft bills | Gmail trigger |
| 03 | [BAS Preparation Pipeline](#03--bas-preparation-pipeline) | Pulls quarterly GST data from Xero, checks reconciliation, flags gaps | Cron (quarterly) |
| 04 | [AI Email Triage](#04--ai-email-triage) | Classifies incoming emails with GPT-4o-mini, routes by category, daily digest | Gmail trigger |
| 05 | [Client Onboarding](#05--client-onboarding) | Engagement letter generation, ID verification, folder creation, welcome email | Webhook |
| 06 | [Smart Client Comms](#06--smart-client-communications) | Appointment reminders (SMS/email), deadline alerts, milestone updates | Cron + Webhook |
| 07 | [Strata AGM Preparation](#07--strata-agm-preparation) | Notice generation, lot owner distribution, proxy tracking, quorum calculation | Webhook |
| 08 | [Export Documentation](#08--export-documentation) | Certification checklist, timezone-aware follow-ups, compliance tracking | Webhook |
| 09 | [AI Voice Agent](#09--ai-voice-agent-trades) | After-hours inbound call handling, emergency triage, callback booking | Phone trigger |
| 10 | [Med-Spa Chatbot](#10--med-spa-chatbot) | Lead qualification, contraindication screening, Cliniko booking | Webhook (chat) |

Workflows 01–06 are purpose-built for accounting/bookkeeping firms. 07–10 demonstrate the same n8n patterns applied to other industries.

---

## Prerequisites

Before deploying any workflow, you need:

### n8n instance

A running n8n instance. Self-hosted recommended for Australian data residency.

```bash
# Docker (quickest)
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n

# Or via npm
npm install n8n -g
n8n start
```

For production, deploy on AWS Sydney (`ap-southeast-2`) or Azure Australia East. See the [n8n self-hosting docs](https://docs.n8n.io/hosting/).

### API keys and credentials

Each workflow lists its required credentials. You'll need accounts with:

| Service | What it's used for | Free tier? |
|---------|-------------------|-----------|
| [Xero](https://developer.xero.com) | Accounting ledger (invoices, contacts, BAS data) | Yes (5 connections on Starter) |
| [Mindee](https://platform.mindee.com) | Invoice/receipt OCR | Yes (250 pages/month) |
| [OpenAI](https://platform.openai.com) | Email classification (GPT-4o-mini) | Pay-as-you-go |
| [Gmail](https://console.cloud.google.com) | Email triggers, sending | Yes (Google Cloud project) |
| [Slack](https://api.slack.com) | Team notifications | Yes |
| [Airtable](https://airtable.com) | Logging, tracking, review queues | Yes (limited) |
| [Twilio](https://www.twilio.com) | SMS reminders | Pay-as-you-go |
| [Google Drive](https://console.cloud.google.com) | Folder creation, document storage | Yes |
| [PandaDoc](https://www.pandadoc.com) | Document generation, e-signatures | Free tier available |

Not every workflow needs every service. See the credential list under each workflow below.

---

## How to deploy

### Option A — Import JSON (recommended)

1. Download the workflow JSON file from this repo (e.g., `workflows/01-document-collection.json`)
2. Open your n8n instance
3. Go to **Workflows → Import from File**
4. Select the JSON file
5. The workflow opens in the editor. It will be **inactive** by default.
6. Replace all placeholder credentials (see below)
7. Test manually using **Execute Workflow**
8. Activate when ready

### Option B — REST API (for Claude Code / programmatic deployment)

If you're using Claude Code connected to your n8n instance:

```bash
# Create a workflow
curl -X POST https://your-n8n-instance.com/api/v1/workflows \
  -H "X-N8N-API-KEY: your-api-key" \
  -H "Content-Type: application/json" \
  -d @workflows/01-document-collection.json

# List existing workflows
curl https://your-n8n-instance.com/api/v1/workflows \
  -H "X-N8N-API-KEY: your-api-key"

# Activate a workflow
curl -X POST https://your-n8n-instance.com/api/v1/workflows/{id}/activate \
  -H "X-N8N-API-KEY: your-api-key"
```


## Replacing placeholder credentials

Every workflow uses placeholder credential IDs. Before activating, you need to:

1. **Create the real credentials** in your n8n instance (Settings → Credentials → Add Credential)
2. **Open the workflow** in the editor
3. **Click each node** that uses a credential and select the real credential from the dropdown
4. **Replace placeholder values** in Set nodes and Code nodes — search for `{{` to find them all

### Placeholder reference

| Placeholder | What to replace with |
|------------|---------------------|
| `{{XERO_CRED_ID}}` | Your Xero OAuth2 credential in n8n |
| `{{GMAIL_CRED_ID}}` | Your Gmail OAuth2 credential |
| `{{MINDEE_API_KEY}}` | Your Mindee API token (from platform.mindee.com) |
| `{{OPENAI_CRED_ID}}` | Your OpenAI API credential |
| `{{SLACK_CRED_ID}}` | Your Slack bot token credential |
| `{{AIRTABLE_CRED_ID}}` | Your Airtable personal access token credential |
| `{{AIRTABLE_BASE_ID}}` | Your Airtable base ID (starts with `app`) |
| `{{TWILIO_CRED_ID}}` | Your Twilio credential |
| `{{TWILIO_PHONE_NUMBER}}` | Your Twilio phone number (e.g., `+61400000000`) |
| `{{GOOGLE_DRIVE_CRED_ID}}` | Your Google Drive OAuth2 credential |
| `{{GOOGLE_SHEETS_CRED_ID}}` | Your Google Sheets OAuth2 credential |
| `{{GOOGLE_CALENDAR_CRED_ID}}` | Your Google Calendar OAuth2 credential |
| `{{PANDADOC_CRED_ID}}` | Your PandaDoc API credential |
| `{{PARTNER_EMAIL}}` | Email address of the firm principal/partner |
| `{{DEFAULT_EXPENSE_ACCOUNT}}` | Xero account code for general expenses (e.g., `429`) |
| `{{UPLOAD_LINK}}` | URL where clients upload documents (your portal/form) |

---

## Workflow details

### 01 — Client Document Collection

**What it does:** When a job needs client documents, this workflow sends the client a checklist email and follows up with escalating reminders at Day 3, Day 7, and Day 14. If documents still aren't received after 14 days, it escalates to the engagement partner. All activity is logged.

**Flow:**
```
Webhook trigger → Build checklist → Log to Airtable → Send request email
  → Wait 3 days → Check status → Still pending? → Reminder #1
  → Wait 4 days → Check status → Still pending? → Reminder #2
  → Wait 7 days → Check status → Still pending? → Escalation email (CC partner)
```

**Trigger:** Webhook POST with `clientId`, `clientName`, `clientEmail`, `engagementType`, `documentsRequired[]`

**Credentials needed:** Gmail, Airtable

---

### 02 — Invoice OCR → Xero / MYOB

**What it does:** Watches a Gmail label for incoming supplier invoices. Extracts data (supplier, amount, GST, line items, ABN) using Mindee OCR. Matches the supplier in Xero. Creates a draft ACCPAY bill. Flags new suppliers or low-confidence extractions for human review.

**Flow:**
```
Gmail trigger (label: "invoices") → Has PDF? → Mindee OCR extract
  → Parse response → Confidence > 85%?
    → Yes: Search Xero contact → Found? → Create draft bill → Log → Notify
    → No: Flag for review → Alert bookkeeper
```

**Trigger:** Gmail poll (every 5 minutes, label filter)

**Credentials needed:** Gmail, Mindee (API key), Xero, Airtable, Slack

**Important notes:**
- Bills are always created as DRAFT — a human reviews before authorising
- Mindee free tier handles 250 pages/month
- Confidence threshold is configurable (default 0.85)
- For MYOB instead of Xero: swap the Xero nodes for HTTP Request nodes calling the MYOB API (see `docs/myob-setup.md`)

---

### 03 — BAS Preparation Pipeline

**What it does:** Runs quarterly (or on-demand). For each client, pulls GST data from Xero, checks for unreconciled transactions and suspicious GST codes, builds a pre-BAS checklist, and emails the accountant with a "ready for review" or "gaps found" summary.

**Flow:**
```
Cron (quarterly) → Get client list → For each client:
  → Pull GST report → Pull bank reconciliation status → Check GST codes
  → Build checklist → Append to tracker sheet
  → Ready? → Email accountant (ready / gaps found)
  → Missing items? → Email client
```

**Trigger:** Cron schedule (1st of Jan, Apr, Jul, Oct at 07:00)

**Credentials needed:** Xero, Airtable, Google Sheets, Gmail

**Important notes:**
- This workflow does NOT lodge BAS — there is no Xero API for BAS lodgement
- It prepares all the data so the accountant can review and lodge manually
- Australian financial year quarters: Q1 Jul–Sep, Q2 Oct–Dec, Q3 Jan–Mar, Q4 Apr–Jun

---

### 04 — AI Email Triage

**What it does:** Monitors the firm's main inbox. Classifies each email into one of six categories using GPT-4o-mini: URGENT, NEW_ENQUIRY, CLIENT_DOCUMENT, INVOICE, ROUTINE, SPAM. Routes accordingly — urgent items get Slack alerts, invoices get forwarded to the OCR pipeline, documents go to the collection workflow. Generates a daily digest.

**Flow:**
```
Gmail trigger → Extract content → GPT-4o-mini classify → Parse response
  → Route by category:
    URGENT → Slack alert + label
    NEW_ENQUIRY → Create lead in Airtable + Slack
    CLIENT_DOCUMENT → Trigger workflow 01
    INVOICE → Forward to workflow 02
    ROUTINE → Label for later
    SPAM → Archive
  → Log classification

Daily digest (5pm):
  → Get today's classifications → Build summary → Email partner
```

**Trigger:** Gmail poll (every 2 minutes)

**Credentials needed:** Gmail, OpenAI, Slack, Airtable

**Important notes:**
- GPT-4o-mini is used for cost efficiency — classification is a simple task
- For privacy-sensitive firms, swap OpenAI for self-hosted Ollama (requires n8n's AI nodes)
- The daily digest helps the partner see what came in without checking Slack

---

### 05 — Client Onboarding

**What it does:** When a new client is accepted, runs them through the full onboarding sequence: generates an engagement letter via PandaDoc, creates a client folder structure in Google Drive, sends a branded welcome email, and logs everything.

**Flow:**
```
Webhook trigger → Normalize data → Generate engagement letter (PandaDoc)
  → Create client folder + subfolders (Drive) → Log to Airtable
  → Send welcome email → Notify team (Slack)
```

**Trigger:** Webhook POST with client details

**Credentials needed:** PandaDoc, Google Drive, Gmail, Airtable, Slack

---

### 06 — Smart Client Communications

**What it does:** Three communication streams in one workflow:
- **Appointment reminders:** 48hr and 2hr SMS/email before scheduled meetings
- **Deadline alerts:** 14/7/2 day notifications for BAS, tax returns, ASIC deadlines
- **Milestone updates:** Automated status emails when work reaches key stages

**Trigger:** Cron (hourly for appointments, daily for deadlines) + Webhook (for milestones)

**Credentials needed:** Google Calendar, Twilio, Gmail, Airtable

---

### 07 — Strata AGM Preparation

**What it does:** Automates the preparation work for strata body corporate AGMs. Generates meeting notices from templates, distributes to lot owners via email, tracks proxy form submissions, calculates quorum requirements, and compiles the final meeting pack.

**Trigger:** Webhook POST with meeting details

**Credentials needed:** Gmail, Google Drive, Airtable

---

### 08 — Export Documentation

**What it does:** Manages export compliance documentation. Maintains a certification checklist per shipment, sends timezone-aware follow-up reminders to overseas contacts, tracks document expiry dates, and flags incomplete documentation.

**Trigger:** Webhook POST with shipment/export details

**Credentials needed:** Gmail, Airtable, Google Sheets

---

### 09 — AI Voice Agent (Trades)

**What it does:** An after-hours inbound call handler for trades businesses. Answers calls when the business owner is unavailable, captures caller details, triages emergencies (e.g., burst pipe vs general enquiry), books callbacks during business hours, and sends an SMS summary to the owner.

**Trigger:** Inbound phone call (via Twilio or VAPI)

**Credentials needed:** Twilio/VAPI, OpenAI, Airtable, Gmail

**Important notes:**
- Requires a voice AI platform (VAPI recommended) connected to n8n via webhook
- The AI agent does NOT give quotes or make commitments — it captures information and books callbacks

---

### 10 — Med-Spa Chatbot

**What it does:** A website chatbot for medical spas and aesthetic clinics. Qualifies leads by asking about treatment interests, screens for contraindications (pregnancy, blood thinners, recent procedures), and books consultations in Cliniko.

**Trigger:** Webhook (chat widget sends messages to n8n)

**Credentials needed:** OpenAI, Airtable, Cliniko API (HTTP Request)

---

## Folder structure

```
n8n-workflows/
├── README.md                          ← you are here
├── workflows/
│   ├── 01-document-collection.json
│   ├── 02-invoice-ocr-pipeline.json
│   ├── 03-bas-preparation.json
│   ├── 04-email-triage.json
│   ├── 05-client-onboarding.json
│   ├── 06-client-communications.json
│   ├── 07-strata-agm.json
│   ├── 08-export-documentation.json
│   ├── 09-voice-agent.json
│   └── 10-medspa-chatbot.json
├── docs/
│   ├── xero-setup.md                  ← Xero OAuth2 setup guide
│   ├── myob-setup.md                  ← MYOB API setup guide
│   ├── mindee-setup.md                ← Mindee OCR setup guide
│   ├── credential-reference.md        ← Full placeholder reference
│   └── australian-gst-codes.md        ← GST/BAS code reference
└── examples/
    ├── webhook-payloads/              ← Sample POST bodies for testing
    │   ├── 01-doc-collection.json
    │   ├── 02-invoice-upload.json
    │   └── ...
    └── airtable-templates/            ← Airtable base templates
        ├── invoice-log.csv
        ├── document-requests.csv
        └── ...
```

---

## Setting up Xero

1. Go to [developer.xero.com](https://developer.xero.com) → My Apps → Add New App
2. App name: anything (can't contain "Xero")
3. Redirect URI: `https://your-n8n-instance.com/rest/oauth2-credential/callback`
4. Copy the **Client ID** and generate a **Client Secret**
5. In n8n: Credentials → New → **Xero OAuth2 API**
6. Paste Client ID and Secret
7. Scopes: `openid profile email offline_access accounting.transactions accounting.contacts accounting.settings accounting.reports.read`
8. Click **Connect** → authorise → select your organisation

**Rate limits:** 60 calls/minute, 5,000 calls/day per organisation. The workflows are designed to stay well within these limits.

**Important:** Xero introduced developer pricing tiers in March 2026. The free Starter tier allows 5 connected organisations with 1,000 API calls/day. If connecting more than 5 client orgs, you'll need the Core tier ($35 AUD/month for up to 50 connections).

---

## Setting up Mindee

1. Go to [platform.mindee.com](https://platform.mindee.com) → sign up (free, no credit card)
2. Go to **API Keys** → **Add New Key** → copy the token
3. In n8n: use **HTTP Request** node with Header Auth
   - Header Name: `Authorization`
   - Header Value: `Token YOUR_API_KEY`

**Free tier:** 250 pages/month. One PDF page = one page. Enough for testing and small practices.

**Endpoints used:**
- Invoice extraction: `POST https://api.mindee.net/v1/products/mindee/invoices/v4/predict`
- Receipt extraction: `POST https://api.mindee.net/v1/products/mindee/receipts/v5/predict`
- Auto-detect (catch-all): `POST https://api.mindee.net/v1/products/mindee/financial_document/v1/predict`

---

## Setting up MYOB (alternative to Xero)

If a client uses MYOB instead of Xero, swap the Xero nodes for HTTP Request nodes. MYOB requires:

1. Register at [developer.myob.com](https://developer.myob.com) → create an app → get API Key + Secret
2. MYOB has **dual authentication**: OAuth 2.0 (your app → MYOB cloud) AND company file credentials (username:password for the specific company file)
3. Every request needs these headers:
   - `Authorization: Bearer {oauth_token}`
   - `x-myobapi-key: {your_api_key}`
   - `x-myobapi-cftoken: {base64(username:password)}`
   - `x-myobapi-version: v2`
4. n8n does NOT have a built-in MYOB node — use HTTP Request with Generic OAuth2 credentials

See `docs/myob-setup.md` for the full setup guide.

---

## Australian GST reference

These workflows handle Australian GST codes. Quick reference:

| Xero TaxType | What it means | Rate |
|-------------|--------------|------|
| `OUTPUT` | GST on Income (sales) | 10% |
| `INPUT` | GST on Expenses (purchases) | 10% |
| `EXEMPTOUTPUT` | GST-Free Income | 0% |
| `EXEMPTEXPENSES` | GST-Free Expenses | 0% |
| `BASEXCLUDED` | BAS Excluded (wages, super, drawings) | 0% |
| `INPUTTAXED` | Input Taxed (residential rent, financial supplies) | 0% |

Always create invoices/bills as **DRAFT** and let a human verify the GST codes before authorising.

---

## Testing

Before activating any workflow:

1. **Replace all placeholders** — search for `{{` in every node
2. **Test manually** — use n8n's "Execute Workflow" button
3. **For webhook workflows** — send a test payload:
   ```bash
   curl -X POST https://your-n8n-instance.com/webhook/doc-collection-start \
     -H "Content-Type: application/json" \
     -d '{
       "clientId": "test-001",
       "clientName": "Test Client Pty Ltd",
       "clientEmail": "your-email@gmail.com",
       "engagementType": "BAS Q3 FY26",
       "documentsRequired": ["Bank statements", "Receipt summary", "Payroll records"]
     }'
   ```
4. **For cron workflows** — click "Execute Node" on the Cron trigger to simulate
5. **Check the execution log** in n8n for errors
6. **Only activate** after manual testing passes

Sample webhook payloads for each workflow are in `examples/webhook-payloads/`.

---

## Troubleshooting

**Xero returns 401:** Your access token expired (they last 30 minutes). n8n should auto-refresh, but if it doesn't, reconnect the credential. If running multiple workflows with the same Xero credential, set workflow concurrency to 1 to avoid refresh token collisions.

**Mindee returns low confidence:** Check the document quality. Scanned PDFs score lower than digital PDFs. Mobile photos need to be well-lit and straight. If confidence is consistently below 0.70, the document type may not be a good fit for automated extraction.

**Gmail trigger not firing:** Make sure the Gmail label exists and emails are being labelled correctly. Check that the credential has the correct scopes (`https://www.googleapis.com/auth/gmail.modify`).

**Airtable "INVALID_REQUEST_UNKNOWN":** Usually means the table name or field name doesn't match. Airtable field names are case-sensitive.

**Slack notifications not sending:** Verify the bot has been invited to the channel. Slack bots can only post to channels they're members of.

---

## Contributing

These workflows are open-source. If you build something useful on top of them or find a bug:

1. Fork this repo
2. Create a branch (`git checkout -b feature/your-improvement`)
3. Commit your changes
4. Push and open a PR

If you've deployed one of these for a real firm and have feedback on what worked or didn't, I'd like to hear about it — open an issue or reach out via [gimhana.dev](https://gimhana.dev).

---

## License

MIT License. Use these however you want. Attribution appreciated but not required.

---

## About

Built by **Gimhana Wijethunga** — Computer Science student at Swinburne University, Melbourne. I build n8n automation workflows for accounting and bookkeeping firms.

Website: [gimhana.dev](https://gimhana-dev.vercel.app/)
Email: scrappynightmare@gmail.com
