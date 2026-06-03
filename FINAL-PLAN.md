# FINAL PLAN: MCA SDR Operating System

Reconciled from plan.md, market-brief.md, strategy-brief.md, the six Pass 3 implementation lenses, and the scope ruling. The scope ruling governs cut and defer decisions except where a compliance blocker or a direct deal-closing function overrides it. Every Build Week 1 item traces to a tactic and a market opportunity. Constraints held throughout: 300 USD per month, one operator maintaining outside work hours, always-on, lean, compliant, 90-day commission run.

## Executive verdict

Proceed. The plan is sound and the warm-lead engine is the right idea, but the single biggest change required is to stop building an agent lab: collapse the three proposed agents into a deterministic n8n-plus-Supabase system on a small VPS, and bar DeepSeek from any PII. Sequence delivery so the operator is closing the existing funded book in week one while the qualification page and UCC engine clear a real and partly overdue compliance gate, because compliance, not engineering, is the true critical path. Judge everything by funded files and protect the prime US call window above all else.

## Market opportunity

Real US MCA origination is about 19 to 20 billion USD at roughly 6 to 7 percent growth, not the 33 to 35 billion that vendor reports claim by blending in revenue-based financing, and the ISO commission pool is only about 1 to 3 billion split across thousands of shops. This is a file-by-file grind where the only durable edge is speed, trust, compliant transparency, and file quality. Demand concentrates in card-heavy, fluctuating-cashflow trades: restaurants, retail, trucking, construction, auto repair, salons, and medical. The seizable opportunity for a solo broker is not net-new demand but better-worked warm demand: the existing funded book and UCC near-payoff merchants who already proved they qualify, gated hard so finite call hours land on the roughly 20x-higher-EV profitable and clawback-survivable merchants.

## Chosen strategy

Win on pre-call intelligence, file quality, and speed, not on factor rate. Work two warm-by-signal pools in order (the funded or repaid book, then UCC near-payoff), funnel every touch to one qualification page that hard-gates bad-fit and distress merchants and captures attorney-approved consent, run a 12 to 15 minute disqualify-loud discovery call that closes into a live document ask, and rank each day's queue by warmth plus EV tier plus clawback survivability. Automate only the deterministic glue in n8n and Supabase on an always-on VPS, and defer every reasoning agent until volume demands it.

## Compliance blockers (resolve before any outreach)

1. **State broker registration confirmed or the state suppressed.** Connecticut was due Oct 1, 2024 and is overdue, so CT is off until confirmed. Clear TX (OCCC, HB 700), NY CFDL, CA DFPI, MO, and the rest. The UCC pipeline enforces a cleared-states allowlist.
2. **Attorney sign-off on the consent checkbox copy before the page goes live.** No SMS and no autodialer ever without it. Copy must name the legal brokerage, enumerate channels (calls including autodialer or prerecorded voice, text, email), state purpose, and revocation, non-pre-checked.
3. **CA SB 362 resolved for the calculator.** In force now. Gate California merchants out of the factor-rate calculator, or add APR output, until counsel signs off.
4. **DNC scrub before the first cold dial,** federal and applicable state, plus a wireless-versus-landline lookup on every UCC number. Manual live-agent dial only on cells. Live-agent voicemail only, no ringless drop.
5. **Call-window enforcement (8:00 to 21:00 merchant local) live and DST-tested** across Israel and US transitions before any automated touch fires.
6. **Consent, DNC, and opt-out logs append-only, US-region, 4-year retention.** Opt-out honored within 10 business days by any reasonable means.

## Build Week 1

The first build sprint. Items 1 to 8 are the warm-book closing motion and can run under an EBR posture with call-window and DNC checks; the operator closes from roughly day 3 to 5. Items 9 and 10 are built in the sprint but go live only after their compliance gates clear (targeted week 2).

1. **Supabase, US-region verified, lean schema** (leads, activities, funder_submissions, pending_actions, workflow_errors, config) with RLS and encrypted phone, email, revenue. Reason: nothing ranks, chases, or stores without it, and region is a one-way door. Serves T6 and T3 to M3 and M5. Lanes: Backend Architect, Data Engineer.
2. **Load the prior funded or repaid book into Supabase.** Reason: fastest cash, no sourcing, no page needed. Serves T1 to M1. Lanes: Data Engineer, Outbound.
3. **Pre-shift ranked-queue view** (Warmth 0.40, EV Tier 0.35, Clawback 0.15, Recency 0.10, minus-4 inside a live clawback window) with a live active-deal join, not a cached flag. Reason: concentrates finite hours on the high-EV merchants. Serves T5 to M5. Lanes: Pipeline Analyst, Backend.
4. **Discovery kit:** the 5-question intake, Tier 1/2/3/Pause thresholds, the live document-ask script, and a templated Tier 1 brief. Reason: disqualifies fast and converts verbal yes into a file on the call. Serves T4 to M3 and M5. Lanes: Discovery Coach, Deal Strategist.
5. **Document engine:** S2-to-S3 stage trigger writes a document-request draft to pending_actions and arms a 48-hour chase, with a pg_cron backup independent of n8n; plus the funder_submissions tracker (stips, cleared, offer_expiry). Reason: plugs the biggest leak between verbal yes and funded file. Serves T6 to M3. Lanes: Pipeline Analyst, Backend, Loan Officer.
6. **Always-on substrate:** small VPS, self-hosted n8n (Docker restart=always), UptimeRobot-to-Telegram, and a config.system_halt kill switch. Reason: the worst failure window is the operator's call window, and a home machine is a single point of failure. Serves all tactics. Lanes: Automation Governance, Optimization.
7. **Model routing and hard caps:** Claude for briefs, note cleanup, and document drafts; DeepSeek only for anonymized non-PII formatting behind a payload-scrubbing node; fail-closed model_blocked flag. Reason: judgment where it matters, PII protected, 300 cap enforced (model spend about 3 to 8 USD). Serves T4 to M3. Lanes: Optimization, Backend.
8. **Opt-out suppression:** a hardcoded keyword check that writes opted_out before any model call, read first by every outbound workflow. Reason: a delayed STOP is a compliance event and cannot depend on a model. Serves the M1 and M4 outreach gate. Lanes: Email Intelligence, Automation Governance, Legal.
9. **Qualification page (build now, live gated by blocker 2 and 3):** distress gate at field 3-4, progressive radio-bucket fields, one state-gated payback calculator (off for CA until SB 362), named non-pre-checked consent, deterministic n8n scoring and routing, and a plain-form fallback if the booking embed fails. Reason: the warming and consent surface for all outbound. Serves T2 and T3 to M2. Lanes: Behavioral Nudge, Growth Hacker.
10. **UCC pipeline as a manual pilot (build now, outreach gated by blocker 1 and 4):** pull, filter to cleared states and the 60 to 100 day window, normalized dedupe (strip legal suffixes before fuzzy match), Apollo enrichment pre-filtered to 25 or fewer, and a static payoff estimator. Reason: the cold engine that ends hostage dependence on company-lead flow. Serves T1 and T4 to M4. Lanes: Data Engineer, Sales Outreach, Legal.

## Phase 2 backlog (ordered by deal impact)

1. **SMS channel** (post-consent, one touch). Highest-converting follow-up. Trigger: consent records flowing and counsel confirms the language names text.
2. **UCC pull automation on a schedule.** Trigger: two clean manual runs.
3. **Inbound reply classifier** (six-label). Trigger: manual reply handling exceeds capacity.
4. **Command-brief and end-of-day report automation; after-call note-cleanup polish.** Trigger: operator time pressure.
5. **Referral ask at funding.** Trigger: first funded deals (week 3 and after).
6. **Local Ollama model and People Data Labs fallback.** Trigger: model-spend or Apollo-coverage pressure.
7. **LinkedIn signal monitoring; aged-lead vendors only if both warm pools run dry** (months 2 to 3).

## Cut

- **Three-agent architecture.** No reasoning load at 10 to 30 touches a day; deterministic n8n plus one Supabase query replaces it. Answers open question 1.
- **Mini PC host.** Single point of failure during the call window. Answers open question 2.
- **DeepSeek for any PII or financial data.** No US residency or DPA; not worth about 2 USD. Answers open question 3.
- **Ringless voicemail.** FCC treats it as a call; live-agent dial only.
- **Aged-lead vendor spend.** Recycled to 5 to 10 brokers at 30 to 40 percent default.
- **In-window LinkedIn outreach.** Competes with calls in a 90-day window.
- **A/B testing infrastructure.** Unreadable at solo volume; sequential two-week directional runs instead.
- **Separate funding-readiness check.** Folded into the one calculator.
- **30-minute consultative call and Sandler emotional-stakes block.** Wrong for a transactional MCA call; 12 to 15 minutes.
- **Automated Tier 3 nurture drip.** A single 90-day re-ping entry instead.
- **Engineering gold-plating:** event-sourcing or CQRS, real-time subscriptions, normalized stip and consent tables, data-lake layers, embedding search and thread reconstruction on replies, LLM-as-judge, cross-provider shadow testing.

## Conflicts resolved (toward the leaner option that still closes deals)

- Aged-lead vendors usable (Outbound) vs toxic (Investment): cut as toxic.
- Stacked re-engagement as an underworked pool (Trend) vs avoid (Investment, Loan Officer): pursue only the clean near-payoff single-position subset.
- Where the 5-question intake lives, page (Behavioral Nudge) vs call (Discovery Coach): both, the page captures structured buckets and pre-populates the brief, the call verifies and deepens.
- Document ask timing, live on call (Discovery Coach, Pipeline) vs automated chase: the live ask is first, the 48-hour automation is the follow-on, not the start.

## Unresolved conflicts for the operator to decide

1. **UCC cold cells:** treat a UCC-sourced sole-proprietor cell as callable by manual live dial after wireless-scrub and DNC (if counsel confirms ordinary consent suffices), OR restrict UCC cold to email and landline only until the merchant opts in on the page. Gates whether UCC is a calling channel at all.
2. **Apollo tier:** stay on the free tier with an aggressive pre-filter to 25 enriched UCC leads a month, OR pay about 49 USD a month for roughly 600 lookups to scale UCC volume. Both fit the cap; the trade is volume versus spend.
3. **California:** gate CA merchants out of the calculator entirely (lose CA volume), OR build an APR-inclusive calculator variant for CA pending SB 362 sign-off (more build and legal time).
4. **Booking tool:** Cal.com (if free-tier booking webhooks are confirmed) vs Calendly. Lock before the page is built, since the booking-created webhook is the strongest warmth signal.
5. **Supabase tier:** if the free tier lacks pg_cron or pgsodium, pay Supabase Pro (about 25 USD) for the backup chase and column encryption, OR implement the backup as a second always-on process and encrypt in n8n.

## Facts to verify (consolidated across all passes)

**Legal (counsel must confirm before launch):**
- TCPA standard for a manual live dial to a UCC-sourced sole-proprietor cell (ordinary vs prior express written consent); the live-agent-voicemail vs ringless line.
- Federal and state DNC scrub and B2B exemption applicability per outreach state; EBR exemption scope for the funded book (cells, state-specific).
- CA SB 362 trigger from the calculator output; NY CFDL calculator-as-offer; broker registration vs disclosure-only per state (CT overdue, TX HB 700, CA DFPI, NY, MO, FL, GA, KS, VA, UT).
- TCPA opt-out 10-business-day revocation rule applicability to B2B.

**Economics:**
- Clawback window by funder (60 vs 90 days vs deal-specific) and the commission basis and any caps. These set the EV-tier weights and the queue penalty.
- Default rate by industry vs the 15 to 20 percent blended average; the payoff percentage at which the original funder re-solicits.

**Sourcing and tools:**
- Top MCA funder names on UCC-1 filings in TX, FL, NY, CA; a sub-100 USD UCC vendor with a parseable secured-party field; Apollo free-tier credit math and lookup key; PDL fallback limits.

**Infrastructure:**
- Supabase US-region options, and pg_cron and pgsodium availability on the free tier; Supabase row-change webhook reliability into n8n; Cal.com free-tier booking webhook; Anthropic hard spend-cap enforcement; DeepSeek retention and DPA status.
