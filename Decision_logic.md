## Decision Logic — Halcyon Health Ticket Triage Engine

The prompt explicitly tells the LLM not to return an action field. The Code node computes it instead, from primary_category, risk_signals, and confidence — using fixed, readable rules (see taxonomy.md for the exact rule order).

This is the central design decision of the whole project, and it's worth being explicit about what it buys: if the escalation behaviour is ever wrong — too cautious, not cautious enough, missing an edge case — that's a rule I can read, test, and fix directly. If the model were deciding action itself inside the prompt, the same bug would be buried in natural-language instructions I'm hoping it interprets consistently on every single run. A rule is auditable in a way a prompt's implicit judgement isn't.

## Why category and risk are separate fields

Covered in detail in taxonomy.md, with a real example from the test data (T027). The short version: category answers "what is this about," risk answers "does this need a human regardless of what it's about," and collapsing them into one label means one of those two questions silently loses.

## Why the taxonomy avoids emotional language

Risk signals in this system are framed in concrete, factual terms — "reports running out of medication," "already changed their own dosing," "reports chest tightness" — rather than emotional tone ("sounds distressed," "seems anxious"). This was a deliberate scope decision: emotional-tone classification is a harder, more subjective problem, and conflating it with the administrative/clinical risk signals this system is actually built to catch would have made both weaker. A patient's emotional state matters, but it's a different problem from "is there a medication gap," and this system is scoped to the latter.

## The Airtable dedup mechanism, and why it's safer than a manual flag

Tickets are picked up by an Airtable Trigger watching a live-filtered view (New Tickets, filtered to Status = New). Once a ticket's status changes, it structurally leaves that view — the trigger literally cannot see it again, because Airtable itself is filtering it out, not because a checkbox somewhere says "already handled."

This is a meaningfully different (and better) design than Project 1's Enrichment Status checkbox gate: a checkbox can be left unset by mistake, or a workflow bug could skip the step that sets it. A view-based gate has no equivalent failure mode — there's no separate "did I remember to flag this" step to forget.

## Why Airtable and HubSpot play different, deliberate roles

Airtable is both the intake point and the Support Desk: tickets are uploaded directly into it, processed in place (an Update, never a Create-then-separately-track), and every ticket's full history — category, risk signals, reasoning, the actual reply, the decision reason — lives on that one record permanently.

HubSpot's contact record, by contrast, only ever holds the latest triage snapshot (last_triage_category, last_triage_urgency) via an upsert that overwrites on every new ticket from the same patient. That's deliberate, not a limitation: a CRM contact record answering "what's this person's current status" is the correct behaviour for a CRM. Full history belongs in the Support Desk record, not duplicated into the CRM.

The fuller audit trail for a given patient's interaction lives as a HubSpot Note attached to their contact record — one note per ticket, carrying the full triage detail (category, risk signals, confidence, reasoning, the actual suggested reply) with its own timestamp. This is why last_triage_date was deliberately dropped as a separate contact property partway through the build: HubSpot's Date-picker property type only accepts a bare calendar day (literally rejects anything with a time-of-day component), so storing a genuine timestamp there would have meant either losing precision or fighting the property type. The note's own hs_timestamp already carries that information at full precision, so a second, lossy copy on the contact record would have been redundant.

## Process Note

Partway through building the HubSpot write-back, Action values sent from n8n (auto_resolve, escalate — lowercase, underscored, matching the Switch node's routing logic) didn't match Airtable's single-select options (Auto Resolve, Escalate — capitalised, matching how a human reads a status board). The fix was to compute both forms in the Code node: the lowercase value stays authoritative for the Switch node's routing, and a separate action_label field carries the human-readable version specifically for the Airtable write. This is a small thing, but worth naming as a real example of a broader principle in this build: internal logic values and user-facing display values are allowed to diverge, and forcing them to match usually means compromising one for the other's sake unnecessarily.

## What I'd build into V2
The IF-node guard for a ticket with no resolvable HubSpot contact (mirroring the known gap documented in Project 2's source-of-truth sync) — this build assumes every sender email can be upserted cleanly, which held for the test data but isn't guaranteed in general.
Genuine sentiment/emotional-state detection as a second, clearly separate field, rather than folding it into risk signals — see "why the taxonomy avoids emotional language" above for why this was deliberately left out of v1 rather than done poorly.
A real accuracy evaluation of the LLM's own classification against the kept-aside answer key (halcyon_health_test_tickets_answer_key.csv), distinct from the separate ML side-experiment's evaluation — see ml-component/ml-component.md for why those two numbers measure different things and shouldn't be confused with each other.