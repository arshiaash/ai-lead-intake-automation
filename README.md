

# AI-Powered Lead Intake & Operations Automation

A complete, production-ready lead management system built from scratch on a zero-cost infrastructure stack. Every new inquiry triggers a fully automated pipeline: AI reads the lead's actual message, writes a personalized follow-up email, scores urgency independently of the lead's own category selection, assigns the right team member, and logs everything across Airtable, Google Sheets, and Slack — without a single human touchpoint.


**Stack**: n8n · OpenAI GPT-5.4-mini · Airtable · Gmail · Google Sheets · Slack · Power BI · HTML/CSS/JS
**Total infrastructure cost**: $0/month (plus ~$0.001–0.003 per lead in OpenAI API usage)

---

## The problem this solves

Most SMBs take 4–8 hours to respond to a new lead. By that point, the prospect has already called someone else. Manual intake processes — checking inboxes, copying data into spreadsheets, sending templated replies — are slow, error-prone, and invisible: nobody knows how many leads fell through the cracks because there's no audit trail.

This system eliminates all of that. Response time drops from hours to under 60 seconds, automatically, 24/7. Every reply reads the lead's actual message, not a template. Every lead, every action, every outcome is logged and queryable. And AI catches urgent leads that the lead's own category selection missed.

---

## Features

**Dual intake paths**
The lead-facing form opens with a choice screen: the structured 5-step form, or a conversational AI chatbot. Both paths produce the same structured payload and hit the same n8n webhook. The chatbot gathers name, email, company, message, and inquiry category through natural conversation, classifies the category itself rather than asking the lead to pick from a dropdown, and only submits once all required fields are confirmed — no partial leads.

**AI urgency detection, independent of self-reported category**
A second OpenAI call reads the lead's actual message text and classifies urgency as Low, Medium, or High — independently of whatever dropdown category the lead selected. A lead who clicks "General Inquiry" but writes "this is urgent, we are losing customers every single day" gets correctly flagged High Priority in Slack, even though their category selection would normally put them at the bottom of the queue. Both the urgency level and the AI's plain-English reasoning are saved to Airtable.

**Rule-based assignment**
Every lead is automatically routed to a team member based on inquiry type: Pricing → Sarah Chen, Partnership → James Okafor, Technical Support → Priya Patel, High-Value Prospect → Founder. Calculated in a single Edit Fields node using a JavaScript ternary expression — no Switch node, no disconnected branches.

**After-hours detection (IST timezone-corrected)**
Timestamps arrive as UTC ISO strings. A JavaScript expression converts to IST (UTC+5:30) before checking business hours (9am–6pm), so the IsAfterHours flag is accurate for a team working Indian Standard Time — not just the raw UTC hour, which produces wrong results for anyone outside UTC+0.

**Repeat contact detection**
Before creating a new record, the pipeline searches Airtable for any existing row matching the same email AND inquiry type. A genuine repeat goes to a lightweight path: the existing record is updated with appended message history and Follow-up Received status, Slack gets a specific repeat-contact alert, and the full AI pipeline is skipped to avoid sending a duplicate email. Matching on email alone would incorrectly flag a person's second, genuinely different inquiry as a repeat — the combined filter handles this correctly.

**Parallel audit log**
The Google Sheets logging branch connects directly off the Webhook node, not off any downstream node. The audit log fires independently of whether the AI, Gmail, or Slack steps succeed. If the main pipeline ever fails partway through, the Sheets log still has a record of the incoming lead.

**3-day follow-up nudge**
A separate scheduled workflow runs daily at 9am, finds any lead still at New status after 3+ days, sends a nudge email, alerts Slack, and updates status to Contacted — preventing leads from silently going cold.

**Error handling**
If anything in the main pipeline fails, a dedicated Error Handler workflow fires and posts an immediate Slack alert naming the failed node and error message. No lead silently disappears.

**Analytics dashboard with AI briefing and Q&A chatbot**
pipeline-control.html is a standalone HTML dashboard pulling live data from Airtable's API. It includes an AI-generated daily briefing (GPT-5.4-mini reads the full pipeline snapshot and writes a 2-3 sentence plain-English summary), a Q&A chatbot grounded in real lead data (ask "who are the high-urgency leads assigned to Sarah Chen?" and get an answer pulled from actual records), and 9 charts covering volume, hours, inquiry types, urgency, workload, status funnel, after-hours split, repeat rate, and a searchable full table.

---

## Project structure

```
lead-intake-form.html                    Lead-facing form + AI chatbot
pipeline-control.html                    Analytics dashboard + Q&A chatbot
workflow-lead-intake-core-pipeline.json  Main n8n automation workflow
workflow-3day-followup.json              Scheduled 3-day nudge workflow
workflow-error-handler.json             Error handler workflow
airtable-powerbi-connector.m            Power Query M script for Power BI
```

---

## Setup

**Prerequisites**: Docker Desktop, Airtable account (free), OpenAI account (~$5 credit covers extensive testing), Google account, Slack workspace (free).

**1. Run n8n locally**

```
docker run -d --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

Open http://localhost:5678 and create your owner account.

**2. Set up Airtable**

Create a base called Lead Management with a table called Leads. Fields needed: Name (text), Status (single select: New/Contacted/In Progress/Converted/Lost/Follow-up Received), Company (text), Email (email), InquiryType (single select: Pricing/Partnership/Technical Support/High-Value Prospect/General Inquiry), Message (long text), Submitted At (date+time), LeadID (autonumber), AssignedTo (text), IsAfterHours (single select: Yes/No), UrgencyLevel (single select: Low/Medium/High), UrgencyReason (text), IsRepeatContact (single select: Yes/No).

Important: type field names carefully and avoid trailing spaces. They are invisible in Airtable's UI but present in the API response, causing silent null values in all downstream integrations.

Create a Personal Access Token at https://airtable.com/create/tokens with scopes: data.records:read, data.records:write, schema.bases:read.

**3. Import n8n workflows**

In n8n, use the "..." menu → Import from file → select each JSON file. Import all three. Configure credentials inside n8n for Airtable (Personal Access Token), Gmail (OAuth2), Google Sheets (OAuth2), Slack (Bot Token), and OpenAI (API Key). Connect the Error Handler: in the main workflow's Settings → Error Workflow → select Lead Intake — Error Handler.

**4. Configure the intake form**

Open lead-intake-form.html in a text editor. Replace WEBHOOK_URL with your n8n webhook's Production URL (visible in the Webhook node after publishing), and set OPENAI_API_KEY to your real key.

**5. Configure the dashboard**

Open pipeline-control.html in a text editor. Fill in the CONFIG block with your Airtable Base ID, Personal Access Token, and OpenAI API key. Open the file in any browser — no server needed.

**6. Connect Power BI (optional)**

Open Power BI Desktop → Get Data → Blank Query → Advanced Editor. Paste the contents of airtable-powerbi-connector.m, replacing the token placeholder with your real Airtable token. Select Anonymous authentication when prompted.

---

## What I actually learned building this

**$json is always relative, never fixed.** In n8n, $json refers to whatever node feeds directly into the current one — not a stable reference. Every time a new node was inserted into an existing chain, downstream expressions broke silently and had to be rewritten as explicit $('Node Name').item.json.body.fieldName references. This was the single most repeated source of bugs throughout the build.

**Invisible trailing spaces in Airtable field names.** Two Airtable fields had been created with a trailing space in their name — completely invisible in Airtable's UI, but present in the raw API response. Both n8n field mappings and Power BI's Power Query returned null silently. Diagnosed by dumping Record.FieldNames() in Power Query and a diagnostic AllFieldNames expression in the dashboard. Would never have been found by looking at the UI.

**"Always Output Data" is not the default.** n8n's Airtable Search node stops execution silently when zero records are returned, rather than passing an empty item forward for the IF node to evaluate. This looks identical to a broken connection on the canvas. The fix is enabling "Always Output Data" in the node's Settings tab — but it's not obvious from the behavior, which just shows the workflow stopping after two nodes with no error.

**OpenAI Responses API shape differs from Chat Completions.** GPT-5.4-mini uses /v1/responses with output[0].content[0].text, not /v1/chat/completions with choices[0].message.content. Code built against the older shape returns 400 errors with no explanation of the actual mismatch.

**Google OAuth tokens expire in testing mode.** The Google Cloud OAuth app stays in testing status (submitting for verification is a lengthy process not worth doing for a portfolio project). This means Gmail and Sheets credentials need manual reconnection periodically — expected behavior, but worth knowing when the "credential needs reconnecting" error appears.

---

## Design decisions

**Custom HTML form over Typeform**: Typeform's free plan now caps at 10 responses/month — unusable for generating realistic demo data. Building the form directly also produced a stronger artifact: the live response clock, choice screen, and conversational chatbot are entirely custom rather than recognizably template-based.

**Self-hosted n8n over n8n Cloud**: n8n's permanent free cloud tier no longer exists as of 2026. Self-hosting via Docker runs on your own machine at zero ongoing cost with full feature access.

**Rule-based assignment over Switch node**: A Switch node creates one output branch per category. Since every lead needs the same downstream processing regardless of category, all branches would need to reconnect to the same next node — fragile and easy to break when inserting new nodes. A single Edit Fields node with a JavaScript ternary expression achieves identical routing in one node with no risk of disconnected branches.

**Email + inquiry type matching for repeat detection, not email alone**: Matching on email alone would flag any second inquiry from the same person as a repeat, even if it's a genuinely new topic deserving full pipeline treatment. The combined filter correctly distinguishes "followed up on the same unresolved issue" from "reached out again about something different."

---

The fictional business "Solis & Pier Advisory" is used throughout as demo context. No real business data is stored or processed.

---
