# SDR Operating System Plan

A lean, AI-assisted operating system for a solo SDR selling US small-business funding. This is the plan under audit. Items marked **settled** are fixed constraints. Items marked **proposal** are open for the audit to challenge.

## 1. Context

- Operator: a solo SDR at a US small-business funding / merchant cash advance (MCA) brokerage, on a 90-day commission-based run.
- Location and hours: works from Israel on US hours, roughly 17:00 to 02:00 Israel time, about 10:00 to 19:00 US Eastern.
- Products in scope: MCA, working capital, business line of credit, term loans, equipment financing.

## 2. Goal and success measure

The single objective is to maximize funded MCA deals. Everything is judged by whether it produces more qualified conversations and more funded files.

A working day is successful if, at the start of the shift, the operator has: a clean ranked call list, the single best next action, warm leads at the top, follow-ups due surfaced, bookings and calendar items visible, and no lead sitting without a next action.

## 3. Hard constraints (settled)

- Total tooling budget is 300 USD per month, not a dollar over.
- Built and maintained by one person, outside working hours. Anything that breaks during work hours costs live selling time.
- The system must be always-on for event handling and scheduled briefs.
- It must not become an overbuilt multi-agent lab. Lean beats clever.
- Outreach must be compliant and non-deceptive. No spam, no fake accounts, no manipulative posting.

## 4. Lead and market approach

Priority order of lead sources:

1. Company-provided leads.
2. Warm re-engagement of old leads.
3. Self-sourced leads when company flow is thin: UCC filings, aged-lead vendors, data providers, LinkedIn.
4. Inbound from the qualification page (see section 5).
5. Referral paths.

This is not a cold-spam machine. Self-sourced cold contacts are warmed before a human call wherever possible.

## 5. The warm-lead engine (core idea)

The operator's own outbound drives contacts to a single smart qualification page. The page self-qualifies and warms them, and warm or booked leads rank to the top of the daily call queue. The page is a capture, warming, and consent surface downstream of outbound, not a traffic source.

How the page warms intelligently:

- Progressive qualification: each step is a micro-commitment and a data point (business name, contact, phone, industry, monthly revenue, time in business, funding need, urgency, use of funds, existing debt, permission to contact).
- Value in exchange: an MCA payback calculator and a funding-readiness check, which lift completion and intent. The calculator makes clear that factor rate is not APR.
- Behavioral signals become the warmth score: form completed, real phone entered, callback requested, time booked.
- Routing: a high score routes straight to a booking link (Cal.com or Calendly). A self-qualified booked call is the warmest exit.

## 6. Sales workflow

- Tiering: leads sorted into Tier 1 (call today), Tier 2 (call or email after Tier 1), Tier 3 (nurture), Pause (bad fit or duplicate).
- Lead briefs: a short brief per Tier 1 lead with why-call, likely funding angle, questions to ask, risk flags, and a suggested opener.
- After-call note cleanup: the operator pastes rough notes and gets a structured CRM record plus a document-request draft.
- Document chase and follow-up discipline so nothing warm goes cold.
- No-answer logic: voicemail or short email, timed re-call for Tier 1, soft follow-up later, then nurture after several touches.
- Work modes the dashboard switches between: Call, Follow-up, Admin, and Meeting, depending on the US call window and what just changed.

## 7. Proposed architecture (proposal, to be tested by the audit)

A three-agent design:

1. **Warm Lead Engine**: sources and warms leads, manages the qualification page and booking routing, scores warmth.
2. **Sales Intelligence Agent**: ranks leads, writes briefs and openers, cleans after-call notes, sets next-best sales action.
3. **Operator / Secretary Agent**: watches inbox, calendar, bookings, and CRM; runs the dynamic next-action dashboard (Do Now, Do Next, Do Later); raises alerts and a daily brief and an end-of-day report.

## 8. Proposed stack (proposal, to be tested)

- **Supabase** as the source-of-truth database (settled choice of store).
- **n8n** as a thin connector layer only: webhooks, triggers, scheduled jobs, moving data between tools. Not the brain.
- **Hermes** agents as the reasoning brain.
- **DeepSeek** as the default model, with Claude or OpenAI reserved for high-judgment or compliance-sensitive work.
- **Cal.com or Calendly** for booking.
- Always-on host: a mini PC or a small VPS (open question).

## 9. Daily operating rhythm

- Before shift: agents prepare warm leads, tiers, briefs, the follow-up queue, and the dashboard.
- Start of shift: review the command brief (do now, do next, hot alerts, meetings, Tier 1 leads, approvals).
- Prime US call window: call Tier 1, process no-answers, send immediate follow-ups, log rough notes.
- Admin and follow-up blocks: approve emails, chase documents, send booking nudges, prepare booked calls.
- End of shift: stats, open loops, tomorrow's priority queue, bugs.

## 10. Budget (settled cap: 300 USD per month)

Indicative allocation: model API spend the largest line, then enrichment or research, then booking, page, and analytics tooling, with a buffer. Hard rules: DeepSeek by default, a local model for cheap formatting where sensible, expensive models only for judgment, event-based rather than always-running agents, and billing caps set wherever possible.

## 11. Compliance posture

Language guardrails: avoid "guaranteed approval," "cheap money," "no risk," or calling factor rate interest. Prefer "approval depends on underwriting," "offer depends on a complete file," "factor rate determines total payback and is not APR."

To verify before any outreach: TCPA and DNC rules for calling and texting US businesses, state-level commercial financing disclosure and broker registration requirements, and consent capture on the page.

## 12. Open questions for the audit

1. Three agents, or fewer? Where do boundaries overlap?
2. Always-on host: mini PC at home, or a small VPS?
3. Is DeepSeek-first acceptable given US business PII and eventual bank statements, or should the default model change?
4. How lean should the week-one build be to start closing fastest?
5. How should warmth scoring be designed so it runs on real signals, not guesses?
6. Which self-sourcing channels are worth the operator's time first?
7. What few metrics belong in the first dashboard?
8. What should be automated only after manual validation?
9. What is missing from this plan?
