# n8n Automation Portfolio

Production-grade n8n workflows demonstrating real automation patterns for SMB and mid-market teams. Each workflow ships as importable JSON + a README that covers the use case, node-by-node logic, required credentials, error handling, and cost expectations.

## Workflows

| Workflow | Vertical | Trigger | AI integration |
|---|---|---|---|
| [Real Estate Lead Qualifier](real-estate-lead-qualifier/) | Real estate / SMB sales | Webhook (Typeform-shaped) | Claude API for lead classification → 3-way branching |

More workflows coming: e-commerce order routing, AI content distribution, CRM churn-risk detection.

## Why these patterns

Most n8n template repos are toy demos that fall over in production. The workflows here implement patterns I'd ship to a paying client:

- **AI fallback rules** — if the LLM classification call fails, rule-based logic still routes the lead correctly
- **Centralized error handling** — every send failure pings a Slack ops channel
- **Credential isolation** — every API key referenced via `{{ $credentials.X }}`, never hardcoded in the JSON
- **Idempotency hints** — Airtable writes use deterministic record IDs where possible
- **Cost-aware LLM calls** — uses Claude Haiku for classification (cheap, ~$0.0003/lead at scale), reserving Sonnet/Opus for tasks that genuinely need deep reasoning

## How to use these

1. Pick a workflow folder
2. Read its README to understand what it does and what credentials it needs
3. Import `workflow.json` into your n8n instance (Cloud or self-hosted)
4. Configure credentials per the `Required credentials` section in each README
5. Test with the provided `sample-payload.json` (where applicable)
6. Customize the prompts and branching to fit your specific intake/routing rules

## Stack

- **n8n** (Cloud or self-hosted Hostinger VPS, ~$5/mo)
- **Anthropic Claude API** — primary LLM for classification + content generation
- **Airtable** — leads / orders / queue storage
- **Twilio** — WhatsApp / SMS (Real Estate)
- **Gmail / Slack / Notion** — notifications + logs

## License

MIT — use, modify, resell. Attribution appreciated, not required.

## About

I'm Andrii Vozniuk, a Senior QA Automation Engineer with 9 years of production experience (currently at N-iX, Kyiv). I work at the intersection of QA reliability practices and modern automation — n8n workflows, LLM-integrated test infrastructure, and CI pipelines that don't break at 2am.

- GitHub: https://github.com/aovozniuk1
- Other portfolios: [QA frameworks](https://github.com/aovozniuk1/portfolio-qa) · [Python dev / FastAPI / RAG](https://github.com/aovozniuk1/portfolio-dev)
