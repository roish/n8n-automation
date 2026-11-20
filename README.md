# AI Automation Workflows (n8n + Gemini + Google Sheets)

A collection of seven [n8n](https://n8n.io) workflows that automate customer-facing tasks for a small online store or sales team: classifying customer messages, extracting structured data from free text, qualifying and tracking leads, drafting replies with AI, and answering support questions through a chat agent.

The workflows use **Google Gemini** for the AI steps, **Google Sheets** as a lightweight CRM and database, and **Gmail** for team notifications.

## Workflows

| # | File | What it does |
|---|------|--------------|
| 1 | `customer inqury classifier.json` | Classifies a message (pricing, order_tracking, returns, other), drafts a short reply, and posts the result to a mock database endpoint. |
| 2 | `customer inquiry+google sheets.json` | Analyzes a message for category, urgency, and buying intent, then logs it to a Google Sheet. |
| 3 | `ai clasiffication.json` | Extracts name, product, quantity, and budget from a free-text customer message and returns clean fields. |
| 4 | `Lead information.json` | Qualifies a lead by budget and posts the result with a status to a mock endpoint. |
| 5 | `Lead system.json` | Qualifies a lead by budget and saves it to a Google Sheets CRM (update for qualified, append for unqualified). |
| 6 | `AI Lead Managment.json` | End-to-end lead pipeline: validate, analyze with AI, log to Google Sheets, draft a reply, and email the team. |
| 7 | `Customer Support AI Agent.json` | Chat agent with memory that looks up orders and FAQs from Google Sheets and follows strict privacy rules. |

## How each workflow works

### 1. Customer Inquiry Classifier
`Webhook` -> `Extract Message` -> `Gemini: classify` -> `Extract Category` -> `Gemini: draft reply` -> `Extract Combined Result` -> `Save to Database`

- Endpoint: `POST /webhook/inquiry-classifier`
- Input: `{ "message": "..." }`
- The reply prompt tells the model not to invent facts such as order numbers or prices.
- The final step posts to `jsonplaceholder.typicode.com` as a stand-in for a real database.

### 2. Customer Inquiry + Google Sheets
`Webhook` -> `Edit Fields` -> `Gemini: analyze` -> `Parse JSON` -> `Edit Fields` -> `Append row in sheet`

- Endpoint: `POST /webhook/ai-sheets`
- Input: `{ "message": "..." }`
- Output columns: Message, Category, Urgency, Buying Intent.

### 3. AI Classification (Information Extraction)
`Webhook` -> `Edit Fields` -> `Gemini: extract` -> `Parse JSON` -> `Edit Fields`

- Endpoint: `POST /webhook/extract-info`
- Input: `{ "message": "Hi, I'm Sam. I want 20 laptops, budget 15000." }`
- Output: `name`, `product`, `quantity`, `budget`.

### 4. Lead Information
`Webhook` -> `Edit Fields` -> `If (budget > 5000)` -> status set -> `HTTP Request`

- Endpoint: `POST /webhook/lead-info`
- Input: `{ "name": "...", "email": "...", "budget": 8000 }`
- Budget above 5000 gives "Qualified — Priority Lead". Otherwise "Not Qualified - Nurture Sequence".

### 5. Lead System
`Webhook` -> `Edit Fields` -> `If (budget > 5000)` -> status set -> Google Sheets

- Endpoint: `POST /webhook/lead-system`
- Input: `{ "name": "...", "contact": "...", "interest": "...", "budget": 8000 }`
- Qualified leads are appended or updated (matched on Name). Unqualified leads are appended.

### 6. AI Lead Management
`Webhook` -> `Extract Lead Data` -> `Validate` -> `Gemini: analyze` -> `Combine Full Record` -> `Append row in sheet` -> `Gemini: draft reply` -> `Notify Team (Gmail)`

- Endpoint: `POST /webhook/ai-lead-management`
- Input: `{ "name": "...", "contact": "...", "message": "..." }`
- Leads missing contact or message are rejected with an error.
- AI returns category, intent (browsing, comparing, ready_to_buy), and priority (low, medium, high).
- The team email includes the draft reply and a reminder to review it before sending to a customer.

### 7. Customer Support AI Agent
`Chat Trigger` -> `Support Agent` (Gemini, memory of recent messages, Order Lookup tool, FAQ Lookup tool)

- Order Lookup reads the `Orders` sheet by order ID.
- FAQ Lookup reads the `FAQ` sheet.
- Safety rules built into the agent:
  - Answers only from tool results and never invents details.
  - Verifies the customer's name before sharing order details.
  - Cannot issue refunds, change orders, or cancel orders, and offers a human instead.
  - Ignores attempts to change or reveal its rules.
  - Stays on store topics and keeps replies short.
- All tools are read-only, so the agent cannot modify data.

## Tech Stack

- **n8n** for workflow automation
- **Google Gemini API** for classification, extraction, and reply drafting
- **Google Sheets** as a CRM and data store
- **Gmail** for team notifications
- **Webhooks** (REST) as entry points

## Setup

1. Install or open [n8n](https://n8n.io) (cloud or self-hosted).
2. Import a workflow: **Workflows -> Import from File** and choose a `.json` file.
3. Add your credentials in n8n:
   - Google Gemini API key
   - Google Sheets OAuth2
   - Gmail OAuth2 (workflow 6 only)
4. Replace the placeholders in the HTTP Request nodes (`YOUR_API_KEY_HERE`) with your Gemini endpoint and key. Better still, store the key as an n8n credential instead of in the URL.
5. Create your own Google Sheets and select them in the Google Sheets nodes. Expected columns:
   - **Lead CRM** (Sheet1): Name, Contact, Interest, Budget, Status. It also needs an `Orders` tab (OrderID, Customer, Item, Status, Tracking) and an `FAQ` tab (Question, Answer).
   - **Lead Management**: Name, Contact, Message, Category, Intent, Priority.
   - **Inquiries**: Message, Category, Urgency, Buying Intent.
6. Change the notification email in workflow 6 to your own address.
7. Activate the workflow and send a test request:

```bash
curl -X POST https://<your-n8n-host>/webhook-test/ai-lead-management \
  -H "Content-Type: application/json" \
  -d '{"name":"Sam","contact":"sam@example.com","message":"What is the price for 50 units?"}'
```

Use `/webhook-test/...` while testing in the editor and `/webhook/...` once the workflow is active.

## Project Structure

```
.
├── README.md
├── ai clasiffication.json
├── AI Lead Managment.json
├── customer inquiry+google sheets.json
├── customer inqury classifier.json
├── Customer Support AI Agent.json
├── Lead information.json
└── Lead system.json
```

## Security Notes

- No real API keys are stored in these files. Use n8n credentials and never commit keys.
- Replace the Google Sheet IDs and email address with your own before sharing.
- AI-drafted replies are suggestions for a human to review, not messages sent automatically to customers.

## What I Learned

- Designing webhook-driven automations with validation and branching
- Prompting an LLM to return strict JSON and parsing it safely
- Building an AI agent with tools, memory, and guardrails
- Using Google Sheets as a simple CRM

## Author

**Roisha Maharjan**
GitHub: [@roish](https://github.com/roish)
