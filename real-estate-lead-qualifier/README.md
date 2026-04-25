# Real Estate Lead Qualifier — n8n + Claude

**What it does:** receives lead form submissions via webhook, uses Claude to classify intent (Hot / Warm / Cold based on budget + timeline + notes), routes Hot leads to instant WhatsApp follow-up, Warm leads to email nurture, Cold leads to Airtable nurture list. Logs every lead to a master Airtable base.

**Use case fit:** real estate agencies, mortgage brokers, SaaS lead-gen with consultative sales motions, any SMB doing $500–$10K deal cycles.

---

## Flow diagram

```
                       ┌──────────────────────────┐
   Typeform submit ──▶ │   Webhook (POST trigger) │
                       │   /webhook/lead-intake   │
                       └────────────┬─────────────┘
                                    │
                       ┌────────────▼─────────────┐
                       │   Claude API call        │
                       │   (claude-3-5-haiku)     │
                       │   Classify: Hot/Warm/Cold│
                       └────────────┬─────────────┘
                                    │
                       ┌────────────▼─────────────┐
                       │   Switch on classification│
                       └─┬──────────┬──────────┬──┘
                         │          │          │
                  ┌──────▼─┐  ┌─────▼────┐  ┌──▼─────┐
                  │  HOT   │  │   WARM   │  │  COLD  │
                  └──────┬─┘  └─────┬────┘  └──┬─────┘
                         │          │          │
              ┌──────────▼──────┐   │          │
              │ Twilio WhatsApp │   │          │
              │ + Calendly link │   │          │
              └──────────┬──────┘   │          │
                         │   ┌──────▼─────┐    │
                         │   │ Gmail send │    │
                         │   │ property   │    │
                         │   │ matches    │    │
                         │   └──────┬─────┘    │
                         │          │          │
                         │          │   ┌──────▼─────────┐
                         │          │   │ Airtable upsert│
                         │          │   │ "Nurture" table│
                         │          │   └──────┬─────────┘
                         │          │          │
                       ┌─▼──────────▼──────────▼──┐
                       │  Airtable upsert         │
                       │  master "Leads" table    │
                       │  (always logged)         │
                       └──────────────────────────┘

  Error path (any node failure) ──▶ Slack #ops-alerts
```

## Why this design

| Decision | Why |
|---|---|
| Claude Haiku, not Sonnet | Classification doesn't need deep reasoning. Haiku is ~10× cheaper (~$0.0003/lead at scale) |
| Rule-based fallback if LLM fails | Rule: budget > $500K AND timeline < 3mo = Hot. Workflow doesn't break on Anthropic API outages |
| Twilio WhatsApp (not SMS) | WhatsApp open rates are 4–5× higher in EU/LATAM/India real estate markets |
| Hot → WhatsApp immediately | Speed-to-lead < 5 min beats response speed > 24h by 9× conversion (industry data) |
| Master log Airtable | Lets you analyze classification accuracy + retrain prompts over time |
| Slack alerts on any send failure | Every silent failure is a missed lead; you find out in real-time, not in a weekly report |

## Required credentials

Configure these in n8n → Credentials → New:

| Service | Type | What it's used for |
|---|---|---|
| Anthropic | HTTP Header Auth (`x-api-key`) | Claude classification calls |
| Twilio | Twilio API | WhatsApp send to Hot leads |
| Gmail | Gmail OAuth2 | Email send to Warm leads |
| Airtable | Airtable API token | Master "Leads" table + "Nurture" table |
| Slack | Slack Webhook URL | Error / ops alerts |

**Never hardcode credentials in `workflow.json`** — every reference uses `{{ $credentials.X.field }}` so credentials live in n8n's encrypted store.

## Setup (5 steps)

1. **Import:** Workflows → Import from File → select `workflow.json`
2. **Credentials:** Workflow's Credentials panel → connect each of the 5 credentials above
3. **Airtable bases:** create two tables in Airtable:
   - `Leads`: fields = `name, email, phone, property_interest, budget_usd, timeline_months, notes, classification, llm_reasoning, created_at`
   - `Nurture`: fields = `name, email, phone, last_contact_at, follow_up_due_at`
4. **Test:** trigger the webhook with `sample-payload.json` (curl example below)
5. **Activate:** toggle workflow to "Active" → it now runs on every form submit

### Test webhook

```bash
curl -X POST https://your-n8n.example.com/webhook/lead-intake \
  -H "Content-Type: application/json" \
  -d @sample-payload.json
```

Expected: a row appears in the `Leads` table with `classification = "Hot"` (because the sample payload has high budget + short timeline).

## Customization

| What to change | Where |
|---|---|
| Classification thresholds | The Claude system prompt in node `Claude Classify` |
| WhatsApp message text | Twilio node `Send Hot WhatsApp` → `body` parameter |
| Property match email template | Gmail node `Send Warm Email` → `htmlBody` parameter |
| Airtable field mappings | Each Airtable node → `Mapping` section |
| Add a new branch (e.g., "Strategic Account") | Duplicate the Switch node + add a new condition |

## Cost estimate

| Component | Per 1,000 leads |
|---|---|
| Claude Haiku classification | ~$0.30 |
| n8n Cloud (or Hostinger self-host) | $0 (free tier covers up to 2,500 executions/mo) or $5/mo |
| Twilio WhatsApp (Hot only, ~10% of leads) | ~$5–10 (at $0.05–0.10 per WhatsApp) |
| Gmail send (Warm only, ~30% of leads) | $0 (free tier) |
| Airtable | $0 (free tier covers up to 1,200 records) |
| **Total per 1,000 leads** | **~$5–11** |

Compare: Zapier with similar logic = $50–100/mo just for execution credits.

## Limitations / when NOT to use this

- **Volume > 50K leads/mo:** consider self-hosted n8n + dedicated database; Airtable hits limits around 50K rows
- **Strict GDPR jurisdictions:** add explicit consent-capture and a deletion path before deploying in EU
- **Compliance-heavy verticals (legal, healthcare):** the Claude prompt logs reasoning to Airtable — consider redacting PII before logging
- **Real-time response < 1s required:** webhook → Claude → branch ≈ 2–4s typical; for sub-second use a rule-based pre-filter
