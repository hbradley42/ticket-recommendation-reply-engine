# HubSpot Lead Routing & Qualification Engine

## The problem
Inbound leads arrive from multiple sources with no consistent scoring or
routing — reps were manually deciding who to prioritise. This automation
scores every new contact against ICP criteria and routes it automatically.

## How it works
1. A new contact in HubSpot triggers a webhook to n8n.
2. n8n enriches the contact with firmographic data.
3. A scoring function applies weighted rules (company size, industry fit).
4. Contacts scoring 60+ are flagged and routed to Sales; 30–59 go to a
   nurture track; below 30 are marked disqualified.
5. The score and routing reason are written back to HubSpot so the CRM
   stays the single source of truth, and a Slack alert notifies the rep.

## Architecture
[Insert screenshot or Mermaid diagram here]

## Decision points
See `decision-logic.md` for the reasoning behind thresholds, duplicate
handling, and failure behaviour.

## Tools
HubSpot (workflows, custom properties, API) · n8n (webhook, HTTP Request,
Code, IF nodes) · Slack

## Note on data
All contacts and companies in this repo are fictional, built for
demonstration purposes only.