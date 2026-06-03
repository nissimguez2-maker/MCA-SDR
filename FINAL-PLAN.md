# FINAL PLAN: MCA SDR Operating System

Reconciled from plan.md, market-brief.md, strategy-brief.md, the six Pass 3 implementation lenses, and the scope ruling, then updated with the operator's five tool and outreach decisions. The scope ruling governs cut and defer decisions except where a compliance blocker or a direct deal-closing function overrides it. Every Build Week 1 item traces to a tactic and a market opportunity. Constraints held throughout: 300 USD per month, one operator maintaining outside work hours, always-on, lean, compliant, 90-day commission run.

## Executive verdict

Proceed. The plan is sound and the warm-lead engine is the right idea, but the single biggest change required is to stop building an agent lab: collapse the three proposed agents into a deterministic n8n-plus-Supabase system on a small VPS, and bar DeepSeek from any PII. Sequence delivery so the operator is closing the existing funded book in week one while the qualification page and UCC engine clear a real and partly overdue compliance gate, because compliance, not engineering, is the true critical path. With the operator's tool choices locked (Cal.com, Supabase free tier, paid Apollo, all legal states including California), the monthly run cost lands around 110 to 160 USD, well under the cap. Because the operator works under the legal umbrella of Flash Cash LLC (the parent of FinBiz Funding), state registration, licensing, and the approved compliance program belong to the parent, which shrinks the legal critical path to one step: obtain and follow FinBiz's approved consent language, disclosures, and cleared-states list before any outreach. FinBiz also funds term loans, lines of credit, equipment, and factoring, so a merchant who does not fit MCA can be placed into another FinBiz product instead of lost. Judge everything by funded files and protect the prime US call window above all else.

## Market opportunity

Real US MCA origination is about 19 to 20 billion USD at roughly 6 to 7 percent growth, not the 33 to 35 billion that vendor reports claim by blending in revenue-based financing, and the ISO commission pool is only about 1 to 3 billion split across thousands of shops. This is a file-by-file grind where the only durable edge is speed, trust, compliant transparency, and file quality. Demand concentrates in card-heavy, fluctuating-cashflow trades: restaurants, retail, trucking, construction, auto repair, salons, and medical. The seizable opportunity for a solo broker is not net-new demand but better-worked warm demand: the existing funded book and UCC near-payoff merchants who already proved they qualify, gated hard so finite call hours land on the roughly 20x-higher-EV profitable and clawback-survivable merchants.

## Chosen strategy

Win on pre-call intelligence, file quality, and speed, not on factor rate. Work two warm-by-signal pools in order (the funded or repaid book, then UCC near-payoff), funnel every touch to one qualification page that hard-gates bad-fit and distress merchants and captures attorney-approved consent, run a 12 to 15 minute disqualify-loud discovery call that closes into a live document ask, and rank each day's queue by warmth plus EV tier plus clawback survivability. Channel by temperature: call hot (engaged) leads, email cold leads, and send SMS only after a lead opts in. Automate only the deterministic glue in n8n and Supabase on an always-on VPS, and defer every reasoning agent until volume demands it.

## Compliance blockers (resolve before any outreach)

The operator works under the legal umbrella of Flash Cash LLC (parent of FinBiz Funding). That moves entity registration, state licensing, and the approved compliance framework to the parent. It does not remove the operator's duty to run their own outbound system within that framework. The gates split accordingly.

**Owned by FinBiz / Flash Cash (obtain and follow, do not invent):**

1. **Cleared-states list.** Operate only where Flash Cash is registered, licensed, or exempt to broker or fund. Get the list from FinBiz compliance; the UCC pipeline enforces it as an allowlist. Do not self-register and do not contact a state until FinBiz confirms it is cleared.
2. **Approved consent and disclosure language.** Use FinBiz's approved TCPA consent checkbox copy and any required state commercial-financing disclosures, including the CA SB 362 APR disclosure. Do not draft your own. The calculator shows factor rate plus the FinBiz-approved APR disclosure to California visitors.
3. **Document and deal handoff.** Confirm how funded files and merchant PII must flow into FinBiz's system of record, and whether bank statements must go to a FinBiz portal rather than your Supabase. Follow their data-handling rules.

**Owned by the operator (enforce in your own system regardless of the umbrella):**

4. **No autodialer, no cold SMS, no ringless voicemail.** Cold outreach is manual-dial calls (to hot or engaged leads) and email only. SMS fires only after opt-in.
5. **DNC scrub and wireless-versus-landline lookup before any cold dial,** using FinBiz's process if they provide one.
6. **Call-window enforcement (8:00 to 21:00 merchant local), DST-tested** for the Israel-to-US offset, before any automated touch.
7. **Consent, DNC, and opt-out logs append-only, US-region, 4-year retention;** opt-out honored within 10 business days. Mirror or feed these into FinBiz's records as they require.

## Build Week 1

The first build sprint. Items 1 to 8 are the warm-book closing motion and can run under an EBR posture with call-window and DNC checks; the operator closes from roughly day 3 to 5. Items 9 and 10 are built in the sprint but go live only after their compliance gates clear (targeted week 2).

1. **Supabase free tier, US-region verified, lean schema** (leads, activities, funder_submissions, pending_actions, workflow_errors, config) with RLS, and app-layer AES-256 encryption (key on the VPS) for phone, email, and revenue. Bank statements are stored in **Supabase Storage**, a private US-region bucket with RLS and short-lived signed URLs, never as raw rows in the database. Reason: source of truth, and region is a one-way door. Serves T6 and T3 to M3 and M5. Lanes: Backend Architect, Data Engineer.
2. **Load the prior funded or repaid book into Supabase.** Reason: fastest cash, no sourcing, no page needed. Serves T1 to M1. Lanes: Data Engineer, Outbound.
3. **Pre-shift ranked-queue view** (Warmth 0.40, EV Tier 0.35, Clawback 0.15, Recency 0.10, minus-4 inside a live clawback window) with a live active-deal join, not a cached flag. Reason: concentrates finite hours on the high-EV merchants. Serves T5 to M5. Lanes: Pipeline Analyst, Backend.
4. **Discovery kit:** the 5-question intake, Tier 1/2/3/Pause thresholds, the live document-ask script, and a templated Tier 1 brief. Reason: disqualifies fast and converts verbal yes into a file on the call. Serves T4 to M3 and M5. Lanes: Discovery Coach, Deal Strategist.
5. **Document engine:** S2-to-S3 stage trigger writes a document-request draft to pending_actions and arms a 48-hour chase, with an independent VPS cron watchdog (separate from n8n) that emails the operator on overdue chases. Plus the funder_submissions tracker (stips, cleared, offer_expiry). Reason: plugs the biggest leak between verbal yes and funded file, and the backup does not depend on n8n uptime. Serves T6 to M3. Lanes: Pipeline Analyst, Backend, Loan Officer.
6. **Always-on substrate:** small VPS (6 to 10 USD), self-hosted n8n (Docker restart=always), UptimeRobot-to-Telegram, and a config.system_halt kill switch. Reason: the worst failure window is the operator's call window, and a home machine is a single point of failure. Serves all tactics. Lanes: Automation Governance, Optimization.
7. **Model routing and hard caps:** Claude for briefs, note cleanup, and document drafts; DeepSeek only for anonymized non-PII formatting behind a payload-scrubbing node; fail-closed model_blocked flag. Reason: judgment where it matters, PII protected, model spend about 3 to 8 USD. Serves T4 to M3. Lanes: Optimization, Backend.
8. **Opt-out suppression:** a hardcoded keyword check that writes opted_out before any model call, read first by every outbound workflow. Reason: a delayed STOP is a compliance event and cannot depend on a model. Serves the M1 and M4 outreach gate. Lanes: Email Intelligence, Automation Governance, Legal.
9. **Qualification page with Cal.com booking (build now, live gated by blockers 2 and 3):** distress gate at field 3-4, progressive radio-bucket fields, one payback calculator (factor rate and total payback, plus APR for California visitors per SB 362), named non-pre-checked consent, deterministic n8n scoring and routing, and a plain-form fallback if the Cal.com embed fails. Cal.com is chosen because its free tier fires the booking-created webhook into n8n, the strongest warmth signal. Reason: the warming and consent surface for all outbound. Serves T2 and T3 to M2. Lanes: Behavioral Nudge, Growth Hacker.
10. **UCC pipeline as a manual pilot (build now, outreach gated by blockers 1, 4, 5):** pull, filter to cleared states and the 60 to 100 day window, normalized dedupe (strip legal suffixes before fuzzy match), Apollo paid-tier enrichment (about 49 USD per month) for owner email and phone, and a static payoff estimator. Channel rule: cold UCC leads get email, become hot on engagement, then get a manual call; SMS only after opt-in. Reason: the cold engine that ends hostage dependence on company-lead flow. Serves T1 and T4 to M4. Lanes: Data Engineer, Sales Outreach, Legal.

## Phase 2 backlog (ordered by deal impact)

1. **SMS channel turned on** (post-opt-in, one touch). Highest-converting follow-up. Trigger: page consent live and validated, counsel confirms the language names text.
2. **UCC pull automation on a schedule.** Trigger: two clean manual runs.
3. **Inbound reply classifier** (six-label). Trigger: manual reply handling exceeds capacity.
4. **Command-brief and end-of-day report automation; after-call note-cleanup polish.** Trigger: operator time pressure.
5. **Referral ask at funding.** Trigger: first funded deals (week 3 and after).
6. **Local Ollama model.** Trigger: model-spend pressure.
7. **LinkedIn signal monitoring; aged-lead vendors only if both warm pools run dry** (months 2 to 3).

## Cut

- **Three-agent architecture.** No reasoning load at 10 to 30 touches a day; deterministic n8n plus one Supabase query replaces it. Answers open question 1.
- **Mini PC host.** Single point of failure during the call window. Answers open question 2.
- **DeepSeek for any PII or financial data.** No US residency or DPA; not worth about 2 USD. Answers open question 3.
- **pgsodium column encryption.** In pending deprecation and not recommended by Supabase; replaced by RLS plus app-layer encryption.
- **Calendly.** Webhooks are paid only; Cal.com gives the booking webhook free.
- **Ringless voicemail.** FCC treats it as a call; live-agent dial only.
- **Cold SMS.** Texting a non-consented lead is the top TCPA exposure; SMS is opt-in only.
- **Aged-lead vendor spend.** Recycled to 5 to 10 brokers at 30 to 40 percent default.
- **In-window LinkedIn outreach.** Competes with calls in a 90-day window.
- **A/B testing infrastructure.** Unreadable at solo volume; sequential two-week directional runs instead.
- **Separate funding-readiness check.** Folded into the one calculator.
- **30-minute consultative call and Sandler emotional-stakes block.** Wrong for a transactional MCA call; 12 to 15 minutes.
- **Automated Tier 3 nurture drip.** A single 90-day re-ping entry instead.
- **Engineering gold-plating:** event-sourcing or CQRS, real-time subscriptions, normalized stip and consent tables, data-lake layers, embedding search and thread reconstruction on replies, LLM-as-judge, cross-provider shadow testing.

## Decisions made

1. **UCC contact by temperature:** call hot or engaged leads, email cold leads, SMS only after opt-in.
2. **All legal states in scope, including California:** the calculator carries the SB 362 APR disclosure for CA visitors.
3. **Apollo paid tier (about 49 USD per month)** for owner email and phone enrichment.
4. **Cal.com** for booking, on its free tier, for the free booking-created webhook.
5. **Supabase free tier,** with bank statements in a private US-region Supabase Storage bucket, app-layer encryption on PII columns, and a VPS cron watchdog as the document-chase backup.

### Confirm with FinBiz / Flash Cash compliance (one call, before launch)

Contact: info@finbizfunding.com or +1 (346) 222-4246.

- The cleared-states list (where Flash Cash can broker or fund), so the UCC allowlist matches.
- The approved consent-checkbox language and the CA SB 362 APR disclosure wording to drop into the page.
- How merchant PII, bank statements, and funded files hand off into FinBiz's system of record, and whether statements go to a FinBiz portal instead of your Supabase Storage.
- Whether manual live-dial to an engaged lead's cell, after wireless scrub and DNC, is within their program.

## Facts to verify (consolidated)

**Legal (confirm with FinBiz / Flash Cash compliance; registration and licensing sit at the parent):**
- The cleared-states list and which products are licensed where; manual-dial-to-cell standard for engaged leads after wireless scrub and DNC.
- FinBiz's DNC scrub process and any EBR handling for the funded book.
- FinBiz-approved CA SB 362 APR disclosure wording and consent-checkbox copy; how the page disclosures must read.
- FinBiz's data-handling and document-handoff rules (system of record, bank-statement portal, retention).

**Economics:**
- Clawback window by funder (60 vs 90 days vs deal-specific) and the commission basis and any caps. These set the EV-tier weights and the queue penalty.
- Default rate by industry vs the 15 to 20 percent blended average; the payoff percentage at which the original funder re-solicits.

**Sourcing and tools:**
- Top MCA funder names on UCC-1 filings by state; a sub-100 USD UCC vendor with a parseable secured-party field; the Apollo paid-tier mobile-credit count specifically (free tier confirmed at only 5 mobile credits per month, which is why the paid tier was chosen).

**Infrastructure (resolved items noted):**
- Cal.com free-tier booking webhook confirmed available. Calendly webhooks are paid only.
- Supabase pgsodium confirmed in pending deprecation; using RLS plus app-layer encryption instead.
- Confirm Supabase free-tier Storage limits (1 GB) are sufficient for 90 days of bank-statement PDFs, and US-region selection at project creation.
- Anthropic hard spend-cap enforcement; DeepSeek retention and DPA status (kept out of PII regardless).
