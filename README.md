# AI Customer Support Agent

![n8n](https://img.shields.io/badge/n8n-workflow-FF6D5A?logo=n8n&logoColor=white)
![AI Powered](https://img.shields.io/badge/AI-GPT--4o--mini-412991?logo=openai&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

An n8n workflow that acts as an AI-powered first-line support agent. It receives customer tickets via webhook, searches your knowledge base, checks order history, generates a personalized response with GPT-4o-mini, and either auto-replies or escalates to the right human team — all while logging everything to Google Sheets and Slack.

## What It Does

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐  ┌──────────────────┐
│  🎫 Webhook  │───▶│  📋 Parse    │──┬▶│  📚 Search   │─▶│  🧠 Merge        │
│  /support    │    │  Ticket      │  │ │  Knowledge   │  │  Context         │
└──────────────┘    └──────────────┘  │ │  Base        │  │  (KB + Orders)   │
                                      │ └──────────────┘  └────────┬─────────┘
                                      │ ┌──────────────┐           │
                                      └▶│  📦 Fetch    │──────────▶│
                                        │  Order Hist. │           │
                                        └──────────────┘           ▼
                                                          ┌──────────────────┐
                                                          │  🤖 AI Support   │
                                                          │  Agent (GPT-4o)  │
                                                          └────────┬─────────┘
                                                                   │
                    ┌──────────────────────────────────────────────┘
                    ▼
          ┌──────────────────┐    ┌──────────────────────────────────────┐
          │  📊 Log to       │    │  🔀 Can AI Resolve?                  │
          │  Google Sheets   │    │                                      │
          └──────────────────┘    │  YES ──▶ 📧 Email Reply ──▶ ✅ Slack │
                                  │  NO  ──▶ 🚨 Escalate to Human Team  │
                                  └──────────────────────────────────────┘
```

## Ticket Categories

| Category | Description | Auto-Resolve? |
|----------|-------------|---------------|
| `technical` | How-to questions, bugs, errors | Yes (if KB has answer) |
| `shipping` | Delivery status, tracking | Yes (with order data) |
| `product` | Feature questions, compatibility | Yes (if KB has answer) |
| `general` | General inquiries | Yes |
| `feedback` | Suggestions, compliments | Yes |
| `billing` | Refunds, charges, invoices | Always escalated |
| `account` | Security, password, access | Always escalated |

## Smart Escalation

When the AI can't resolve a ticket, it routes to the right team:

- **#billing-support** — refund requests, payment issues, subscription changes
- **#tech-support** — bugs, errors, technical issues beyond KB
- **#security-urgent** — account compromise, unauthorized access
- **#support-escalations** — everything else that needs a human

Each escalation includes the AI's draft response and internal notes so the human agent can respond faster.

## API Usage

### Send a Support Ticket

```bash
curl -X POST https://your-n8n.com/webhook/support \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sarah Chen",
    "email": "sarah@example.com",
    "customer_id": "cust_12345",
    "account_tier": "pro",
    "channel": "web",
    "subject": "Cannot export reports to PDF",
    "message": "When I click Export to PDF on the Reports page, nothing happens. I am using Chrome on Mac. This started yesterday.",
    "previous_tickets": 2
  }'
```

### Response

```json
{
  "success": true,
  "ticket_id": "TKT-M5K8R2",
  "category": "technical",
  "priority": "medium",
  "resolved_by_ai": true,
  "response": "<p>Hi Sarah, I understand the PDF export issue...</p>",
  "escalated_to": null
}
```

### Supported Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Customer name |
| `email` | string | yes | Customer email |
| `subject` | string | yes | Ticket subject |
| `message` | string | yes | Full message |
| `customer_id` | string | no | For order lookup |
| `account_tier` | string | no | free, pro, enterprise |
| `channel` | string | no | web, email, chat, api |
| `language` | string | no | ISO language code |
| `previous_tickets` | number | no | Prior ticket count |

## Setup

### Prerequisites

- [n8n](https://n8n.io/) (self-hosted or cloud)
- OpenAI API key (GPT-4o-mini)
- SMTP credentials for sending replies
- Google Sheets API access
- Slack workspace with a bot token
- Knowledge base API endpoint (optional)
- Order/CRM API endpoint (optional)

### Installation

1. **Import the workflow** — Copy `workflow.json` and import in n8n

2. **Set up credentials** in n8n:
   - **OpenAI** — your API key
   - **SMTP** — for customer reply emails
   - **Google Sheets** — OAuth2 connection
   - **Slack** — Bot token with `chat:write` scope

3. **Create a Google Sheet** with columns:

   | Ticket ID | Date | Customer | Email | Channel | Subject | Category | Priority | Sentiment | Resolved by AI | Escalated To | Confidence | Tags |
   |-----------|------|----------|-------|---------|---------|----------|----------|-----------|----------------|--------------|------------|------|

4. **Set environment variables:**
   ```
   KNOWLEDGE_BASE_URL=https://your-api.com/kb/search
   ORDER_API_URL=https://your-api.com/orders
   SMTP_FROM=support@yourcompany.com
   GOOGLE_SHEET_ID=your-sheet-id
   SLACK_CHANNEL=#support-resolved
   ```

5. **Create Slack channels:**
   - `#support-resolved` — AI-resolved ticket logs
   - `#support-escalations` — general escalations
   - `#billing-support` — billing team
   - `#tech-support` — technical team
   - `#security-urgent` — security issues

6. **Activate the workflow** — it's ready to receive tickets

### Integration Examples

**Website chat widget:**
```javascript
async function submitTicket(formData) {
  const response = await fetch('https://your-n8n.com/webhook/support', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(formData)
  });
  return response.json();
}
```

**Connect to existing helpdesk:**
Replace the webhook trigger with Zendesk, Freshdesk, Intercom, or HelpScout triggers available in n8n.

## Example Scenarios

### Scenario 1: Auto-Resolved (Technical)

**Customer:** "How do I enable two-factor authentication?"

**AI finds KB article** → sends step-by-step instructions → logs as resolved → Slack notification

### Scenario 2: Escalated (Billing)

**Customer:** "I was charged twice for my subscription this month"

**AI classifies as billing** → drafts empathetic response → escalates to `#billing-support` with draft and order history → Slack alert with full context

### Scenario 3: Urgent Escalation (Security)

**Customer:** "Someone changed my password and I can't access my account"

**AI flags as critical** → immediately escalates to `#security-urgent` → includes all account details for the security team

## Google Sheets Dashboard

| Ticket ID | Customer | Category | Priority | Resolved by AI | Confidence |
|-----------|----------|----------|----------|----------------|------------|
| TKT-M5K8R2 | Sarah Chen | technical | medium | TRUE | 0.92 |
| TKT-N6L9S3 | John Smith | billing | high | FALSE | 0.88 |
| TKT-P7M0T4 | Maria Lopez | shipping | low | TRUE | 0.95 |
| TKT-Q8N1U5 | Alex Kim | account | critical | FALSE | 0.91 |

## Customization

### Add Your Knowledge Base

Replace the "Search Knowledge Base" HTTP Request node with your actual KB API. Works with Notion, Confluence, Zendesk Guide, or any REST API.

### Use a Vector Store (RAG)

Replace the HTTP Request node with n8n's built-in vector store nodes (Pinecone, Qdrant, Supabase) for semantic search over your docs.

### Change the AI Model

Swap `gpt-4o-mini` for `gpt-4o` for complex support scenarios, or use Claude, Gemini, or a local model via Ollama.

### Add More Channels

Connect Intercom, Zendesk, Freshdesk, WhatsApp, or Telegram as additional input triggers alongside the webhook.

### Customize Escalation Rules

Edit the AI system prompt to change when tickets get escalated vs. auto-resolved based on your business rules.

## Cost Estimate

With GPT-4o-mini at ~$0.15/1M input tokens:
- **50 tickets/day** ≈ $0.03/day ($0.90/month)
- **200 tickets/day** ≈ $0.12/day ($3.60/month)
- **500 tickets/day** ≈ $0.30/day ($9.00/month)

## Author

**Ivan Siyanko** — [siyanko.com](https://siyanko.com)

## License

MIT — see [LICENSE](LICENSE)
