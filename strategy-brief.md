# Pass 2 Reconciliation: Strategy Brief

Narrowed from six strategy lenses. This picks the winning play and the tactics that implement it, and parks the alternatives.

## Chosen strategy

Win on pre-call intelligence, file quality, and speed, not on factor rate. Work two warm-by-signal pools in strict order (the prior funded or repaid book first, then UCC near-payoff merchants), drive every touch to one qualification page that hard-gates bad-fit and distress merchants and captures TCPA-scoped consent, then run a 12-15 minute disqualify-loud discovery call that closes into a live document ask. Rank each day's queue by warmth plus EV tier plus clawback survivability so finite call hours go to the roughly 30-50k USD per month, 18-plus month, no-stack merchants. Automate only the deterministic glue (page scoring, routing, document chase) in n8n and Supabase. Do not build a multi-agent lab.

## Key tactics (ranked)

1. **Signal-referencing outreach to the two warm pools, short cadences, single CTA to the page.** [Sales Outreach, Deal Strategist] Re-engagement: 5 touches over 8 days. UCC near-payoff: 5 touches over 14 days. Openers name the specific signal (prior funding month, or UCC funder and filing date) and use a pain question, never a rate quote. Phone, voicemail, email; one voicemail per cadence. Target UCC merchants at 60-70 percent payoff (60-100 day filings) with pull-to-first-call under 5 business days.

2. **One qualification page that gates early, mirrors transactional language, runs one calculator, and captures scoped consent.** [Behavioral Nudge, Growth Hacker] Distress gate at field 3-4 (route a "yes" to a static SBDC referral, no booking). Progressive radio-bucket fields as micro-commitments. One inline payback calculator (factor-rate-is-not-APR, no APR conversion). Named, non-pre-checked TCPA consent checkbox. Booking embed with a plain-form fallback if it fails to load.

3. **Deterministic warmth scoring and routing in n8n, not an agent.** [Growth Hacker, Pipeline Analyst, Behavioral Nudge] Hard-coded point values: score 80-plus routes to the booking link, 50-79 to the SDR queue, under 50 to nurture. Writes the exact fields the queue formula needs. No model call at this volume.

4. **A 12-15 minute disqualify-loud discovery call on a 5-question financial intake that closes into a live document ask.** [Discovery Coach, Deal Strategist] Q1 deposits, Q2 time in business, Q3 stacking, Q4 NSF and negative days, Q5 average daily balance, each mapped to Tier 1/2/3/Pause in real time. Stacking and NSF are hard disqualifiers, stated flatly. Confirm the signer is the economic buyer or do not advance. Request 3 months of statements plus the application on the call or within 10 minutes.

5. **Rank the daily queue by a composite EV and clawback formula, and protect the call window with work-mode blocks.** [Pipeline Analyst] Score = Warmth 0.40 + EV Tier 0.35 + Clawback Survivability 0.15 + Recency 0.10, with a hard minus-4 for any deal inside a live 90-day clawback window. Reserve 10:00-14:00 ET for Tier 1 calls only; command brief ready before shift; enforce the 8:00-21:00 merchant-local call window automatically (Israel-timezone risk).

6. **Wire document collection as a Supabase stage-transition trigger, measured as S3-to-S4 completion rate.** [Pipeline Analyst, Discovery Coach] On S2 exit, n8n writes the S3 timestamp, generates the document-request draft, and sets a 48-hour auto-chase; non-completion fires the chase. The live ask in tactic 4 precedes the automated chase. Makes document velocity a number, not a discipline question.

## Parked or alternative tactics

- **SMS as a parallel channel** [Sales Outreach, Growth Hacker]: blocked until the page is live and consent captured; allow only a single post-consent touch.
- **LinkedIn cadence and aged-lead vendor spend** [Sales Outreach]: cut now (toxic default, scarce solo hours); revisit in months 2-3 only if the two pools run dry.
- **30-day nurtures and automated Tier 3 nurture** [Sales Outreach, Pipeline Analyst]: non-responders go to a 90-day re-ping; Tier 3 called manually in follow-up blocks.
- **Multi-agent Warm Lead Engine and separate scoring or ranking agents** [Growth Hacker, Pipeline Analyst]: collapse to n8n plus one Supabase query until volume demands reasoning.
- **A/B split testing** [Growth Hacker]: use sequential 2-week directional runs at this volume.
- **Separate funding-readiness check** [Behavioral Nudge]: folded into the one calculator.
- **Consultative pitch, Sandler emotional-stakes, 30-minute call** [Discovery Coach]: cut for MCA.

## Constraints to respect

- **Budget:** UCC vendor under 100 USD, enrichment (Apollo free tier), booking, page, Supabase and n8n free tiers. No aged-lead spend, no paid analytics.
- **Compliance (gating):** TCPA one-to-one consent gates all SMS and the page consent copy needs attorney review before any contact. Clear state broker registration and disclosure (VA, TX HB 700, NY, CA and others) per state and filter UCC pulls to cleared states. No hype language. Factor rate is not APR.
- **Solo-operator limits:** protect the prime call window; keep everything deterministic and low-maintenance so nothing breaks during work hours; always-on with fallbacks; low volume means sequential learning, not A/B.

## Conflicts (unresolved, for later steps)

- TCPA one-to-one consent applicability to B2B and sole-proprietor cell calls and texts, especially UCC-sourced contacts with no page opt-in. This gates UCC outreach as a calling or texting strategy. [Deal Strategist, Sales Outreach, Behavioral Nudge, Growth Hacker]
- Stacking tolerance threshold: Discovery Coach uses 2-plus positions as a hard disqualify and one advance under 30-50 percent of deposits as acceptable; actual funder guidelines must override. [Discovery Coach]
- Ringless voicemail is treated as a call under FCC guidance (same consent analysis); unresolved whether to use any voicemail-drop service at all. [Sales Outreach]

## Facts to verify (consolidated)

- TCPA one-to-one consent (Jan 2025 FCC rule, under Eleventh Circuit challenge): does it reach B2B and sole-proprietor cell texting and UCC-sourced cold calls without page opt-in? Does naming a UCC funder and filing date in outreach trigger a state commercial-solicitation disclosure (TX, NY, CA, FL)?
- Clawback window and commission basis for the brokerage's top funders (60 vs 90 days; percent of funded amount; per-deal or per-period caps). These set the EV-tier weights and the queue penalty.
- Payoff percentage at which the original funder re-solicits (60, 70, 80) to time the re-up window.
- Top MCA funder names on UCC-1 filings in TX, FL, NY, CA; a sub-100 USD UCC vendor with debtor-contact export plus an enrichment tool for the owner's direct phone.
- Whether the page's inline calculator output is a commercial financing disclosure that triggers CA SB 1235 or NY CFDL before a formal offer.
- Cal.com free-tier iframe embed and webhook support, and n8n free-tier execution limits for the form-to-score-to-nudge flow within 10 minutes.
- Supabase US-region, and whether column-level encryption (not just row-level security) is needed for bank-statement and revenue or PII fields given DeepSeek access.
