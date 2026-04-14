# AI Email Autoresponder with Human-in-Loop Approval

> An AI system that drafts contextual email replies using RAG, but requires human approval before sending -- balancing speed with governance.

## Scenario

A B2B management consulting firm specialising in digital transformation for mid-market companies.

| Detail | Value |
|--------|-------|
| Industry | Management Consulting |
| Size | 40 consultants, 5 admin staff |
| Email volume | 200+ client emails/day |

## The Problem

aps received 200+ client emails daily across project updates, meeting requests, document queries, and billing questions. Their challenges:

- Average response time: 48 hours (client SLA target: 24 hours)
- Admin staff spent 6+ hours/day writing replies to repetitive questions
- Consultants drafted responses at night, leading to burnout
- No consistency: different staff gave different answers to the same question
- Compliance concern: a junior staff member once sent incorrect contract terms by email

**Pain:** 48-hour response times were costing them clients. 3 prospects cited "slow communication" in lost-deal feedback. But they couldn't just let AI send emails unsupervised -- one wrong answer about contract terms could be legally binding.

## The Solution

An n8n workflow that reads incoming emails, summarises them, queries a knowledge base (RAG) for the right answer, drafts a reply, and sends it to a human approver. The human can approve, edit, or reject before the email goes out.

### How It Works

```
Incoming Email (IMAP)
       |
       v
[Email Trigger] --> [Summarise Email]
       |
       v
[RAG Lookup: Qdrant Knowledge Base]
  |-- Company policies
  |-- Service descriptions
  |-- Pricing templates
  |-- FAQ answers
       |
       v
[AI Agent: Draft Reply]
  |-- Context: email summary + RAG results
  |-- Tone: professional, concise
  |-- Includes: source references
       |
       v
[Human Approval Email]
  |-- "Approve" button --> Send as-is
  |-- "Edit" link --> Modify then send
  |-- "Reject" button --> Flag for manual handling
       |
       v
[Send Approved Email] --> [Log to Audit Trail]
```

### Architecture

| Component | Technology |
|-----------|-----------|
| Orchestration | n8n (self-hosted) |
| Email Integration | IMAP trigger + SMTP send |
| Knowledge Base | Qdrant vector store |
| Embeddings | OpenAI text-embedding-3-small |
| LLM | OpenAI GPT-4o |
| Summarisation | LangChain Summarisation Chain |
| Approval | Email-based approve/reject flow |

## Key Design Decisions

- **Why human-in-the-loop?** Legal risk. An AI sending incorrect contract terms or pricing could create binding obligations. Human approval eliminates this risk while keeping the speed benefit.
- **Why RAG instead of fine-tuning?** The knowledge base changes frequently (new services, updated pricing). RAG lets you update documents without retraining. Drop a new PDF and the answers update instantly.
- **Why email-based approval (not Slack)?** aps's senior consultants live in Outlook. Asking them to check another tool would kill adoption. Approve/reject buttons in email mean zero workflow change.
- **Why audit logging?** Compliance requirement. Every AI-drafted email must be traceable: who approved it, when, what was the original draft vs final version.

## Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Average response time | 48 hours | 3-6 hours (depending on approver availability) | ~87-92% reduction |
| Admin hours on email | 6+ hrs/day | 1-2 hrs/day | ~70-75% reduction |
| Response consistency | Variable | ~80-85% aligned with company docs (improving monthly) | Noticeable improvement |
| Client satisfaction (NPS) | +22 | +38 (measured Q2 vs Q1) | +16 points |
| Compliance incidents | 2/quarter | 0 in first 6 months | Eliminated so far |

> **Note:** Approval rate improved from 35% in week 1 to 78% by week 3 as the system prompt was refined using rejected draft patterns.

## Setup

### Prerequisites
- Docker & Docker Compose
- OpenAI API key
- IMAP email account credentials
- SMTP sending credentials

### Quick Start
```bash
docker compose up -d
# Import workflow/workflow.json into n8n
# Configure IMAP credentials (incoming email)
# Configure SMTP credentials (outgoing email)
# Set OpenAI API key
# Upload knowledge base documents to Qdrant
```

### Environment Variables
| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | OpenAI API key |
| `IMAP_HOST` | Incoming email server |
| `IMAP_USER` | Email account username |
| `IMAP_PASS` | Email account password |
| `SMTP_HOST` | Outgoing email server |
| `APPROVER_EMAIL` | Email address for approval requests |

## Example Run

### Incoming Email

```
From: sarah.williams@brightpath.com
Subject: Re: Pricing for digital transformation assessment

Hi,

Thanks for the initial conversation last week. We're keen to move forward
but need to understand the pricing structure before we can get budget approval.

Could you send over the costs for your assessment and strategy packages?
We're a mid-market retailer with around 300 employees and 12 retail locations.

Also, do you offer any bundled pricing if we commit to both discovery and
strategy phases?

Thanks,
Sarah Williams
CFO, BrightPath Retail
```

### AI Processing

1. **Summarisation:** "Client asking about pricing for digital transformation assessment and strategy packages, including potential bundle discount."
2. **RAG Lookup:** Retrieved pricing tiers from knowledge base -- Discovery (£5,000), Strategy (£15,000), bundled discount (£18,000). Confidence: 0.92.
3. **Draft Generated:**

```
Subject: Re: Pricing for digital transformation assessment

Dear Sarah,

Thank you for your interest in moving forward -- great to hear the initial
conversation was useful.

Our engagement tiers are as follows:

- Discovery (£5,000) -- a 2-week current-state assessment including
  stakeholder interviews and an opportunity identification report.
- Strategy (£15,000) -- a 6-8 week programme covering full process mapping,
  technology assessment, and a prioritised transformation roadmap.

We do offer bundled pricing: if you commit to both Discovery and Strategy
together, the combined cost is £18,000 (a 10% saving vs booking separately).

If a full implementation phase follows, that is scoped and quoted after the
Strategy phase is complete, with pricing starting from £35,000 depending on
the programme's breadth.

I'd be happy to set up a call to walk through which option best fits
BrightPath's needs. Would any time on Tuesday or Wednesday afternoon work?

Best regards,
Axxxx Professional Services
```

### Approval Flow

The designated approver receives an email containing:

```
--- APPROVAL REQUEST ---

Original email from: sarah.williams@brightpath.com
Summary: Client asking about pricing for assessment and strategy packages,
         including bundle discount.

--- PROPOSED REPLY ---

[Draft reply as shown above]

--- ACTIONS ---

  [ Approve ]    [ Edit ]    [ Reject ]

---
```

After the approver clicks **Approve**, the audit log records:

```
{
  "email_id": "msg-20250602-1037-brightpath",
  "action": "approved",
  "approver": "e.richardson@apspro.co.uk",
  "approved_at": "2025-06-02T11:14:32Z",
  "draft_version": 1,
  "edits_made": false
}
```

## Challenges & Iteration

- **V1 approval rate was only 35%.** The AI drafts were too generic -- they sounded like a chatbot, not a consultant. Fix: fine-tuned the system prompt with 50 real email examples from senior consultants. Added tone guidelines ("match the formality of the incoming email"). Approval rate climbed to 78% over 2 weeks.
- **Email threading was a nightmare.** Forwarded emails, reply chains with 10+ messages, and emails with inline images broke the summarisation chain. Fix: added a pre-processing step that extracts the most recent message only, strips HTML, and handles multipart MIME.
- **The RAG knowledge base returned pricing information from 2024 in response to 2025 queries.** An approver caught it before sending but flagged it as a trust-breaker. Fix: added document versioning -- when a new pricing doc is uploaded, the old one is archived (not deleted) and tagged as superseded. Retrieval filters to current versions by default.
- **False urgency detection:** some auto-generated emails (invoice reminders, newsletter unsubs) were being drafted as urgent responses. Fix: added a classification step that filters out automated emails before drafting.

## Constraints & Trade-offs

- **Why email-based approval over Slack:** Tested Slack buttons first. Only 2 of 8 senior consultants checked Slack regularly. Email approval had 4x higher response rate. "Use the channel people already check."
- **Why IMAP over Gmail API:** aps uses Microsoft 365. Gmail API was not an option. IMAP is universal but has quirks (no push notifications, polling delay of 1-2 minutes).
- **Why not auto-send for low-risk emails:** Discussed with aps. Even simple "thank you for your email" responses must be approved. Their compliance officer was firm: "nothing goes out without a human seeing it first." The approval step is a feature, not a limitation.
- **Why Qdrant for the knowledge base:** Same self-hosting requirement as Financial RAG (client data sensitivity). Small corpus (~200 documents) so Qdrant was fine.

## Edge Cases & Error Handling

| Scenario | Handling | Fallback |
|----------|----------|----------|
| Email with attachments | Summarises text body only, notes "X attachments detected" in draft | Approver decides whether attachments are relevant |
| Non-English email | Language detection triggers, draft generated in detected language | If detection fails, defaults to English with a note |
| IMAP connection drops | Retry with backoff (3 attempts), then Slack alert to admin | Reconnects on next polling cycle |
| Approver doesn't respond within 4 hours | Escalation email to backup approver | If still no response by 8 hours, Slack alert |
| Spam/phishing email | Not filtered by this workflow (handled by Exchange spam filter upstream) | Only legitimate emails reach the workflow |
| Email with sensitive keywords (legal, lawsuit, termination) | Flagged as "high sensitivity", routed to managing director instead of standard approver | Separate approval chain |

## Monitoring & Maintenance

- **Daily metrics:** emails processed, drafts generated, approval rate, average time-to-approval, rejection reasons
- **Weekly digest:** top 5 rejected drafts with rejection reasons -- used to improve the system prompt and knowledge base
- Rejected drafts feed a "what the AI gets wrong" log that's reviewed monthly. Common patterns become new RAG documents or prompt refinements.
- **IMAP connection monitor:** heartbeat every 5 minutes, alert on 3 consecutive failures
- **Knowledge base freshness:** quarterly review to archive outdated documents

## Customisation

- **Add category routing:** Route emails to different approvers based on topic (billing -> finance, technical -> engineering)
- **Add SLA alerts:** If no approval within 2 hours, escalate to a backup approver
- **Add sentiment detection:** Flag angry/urgent emails for priority handling
- **Replace email approval with Slack:** Swap the approval email node for Slack interactive buttons

## Tech Stack

`n8n` `OpenAI GPT-4o` `Qdrant` `RAG` `IMAP/SMTP` `Human-in-the-Loop` `Docker`

## Lessons Learned

1. Human-in-the-loop isn't a compromise, it's a feature. It's what made the client trust the system enough to actually use it.
2. The approval step also functions as a training signal: rejected drafts reveal where the knowledge base has gaps.
3. Email-based approval had 4x higher adoption than a Slack-based prototype we tested first.

---

*Built by [Kessog Chan](https://linkedin.com/in/kessogchan) -- AI Solutions | Workflow Automation | Finance & Banking*
