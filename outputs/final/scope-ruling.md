# Scope Ruling

Authority: Senior Project Manager, acting as scope authority. Inputs: strategy-brief.md and all six Pass 3 lenses. Mandate: the smallest version that still closes funded MCA deals under the settled constraints (300 USD per month, one operator building and maintaining outside work hours, always-on, lean not overbuilt, compliant and non-deceptive), on a 90-day commission run.

## Verdict

Almost every tactic in the strategy brief survives, but the build is wrong in two ways the plan did not see, and right to sequence in a way the plan did not state. First, the chosen strategy is sound and I am not cutting its spine. Second, the single biggest scope decision is this: split the system into a warm-book closing motion that can start in days, and a cold engine gated behind a compliance checklist that has live, already-overdue exposure. The operator should be dialing the existing funded book for revenue while the qualification page and UCC engine clear legal review. Do not wait for the full system to close the first deals.

## Compliance gates (blocking, before any outreach of any kind)

These are not build tasks, they are go or no-go gates. The system does not go live against a state until its gate clears.

1. State broker registration confirmed or the state suppressed. Connecticut registration was due Oct 1, 2024 and is already overdue, so CT is suppressed until confirmed. Confirm TX (OCCC, HB 700), NY CFDL, CA DFPI, MO, and the other disclosure states. UCC pulls are filtered to a cleared-states allowlist in the pipeline.
2. Attorney sign-off on the consent checkbox copy before the page goes live. No SMS and no autodialer, ever, without it.
3. CA SB 362 resolved for the calculator. It is in force now. Gate California merchants out of the factor-rate calculator, or add APR output, until counsel signs off.
4. Federal and state DNC scrub completed before the first cold dial, plus a wireless-versus-landline lookup on every UCC number. Manual live-agent dial only on cells. Live-agent voicemail only, no ringless drop service.
5. Call-window enforcement (8:00 to 21:00 merchant local) live and tested across Israel and US daylight-saving transitions before any automated touch fires.
6. Consent, DNC, and opt-out logs are append-only, US-region, 4-year retention.

## Build now (the closing core)

1. Supabase, US-region verified at creation, with the lean schema: leads, activities, funder_submissions, pending_actions, workflow_errors, config (system_halt). Row-level security on all tables; encrypt phone, email, revenue. Source of truth. Closes deals because nothing ranks or chases without it.
2. Load the prior funded and repaid book into Supabase. This is the fastest cash and needs no sourcing and no page.
3. The daily ranked queue as one Supabase view run pre-shift (Warmth 0.40, EV Tier 0.35, Clawback 0.15, Recency 0.10, minus-4 inside a live clawback window). Tells the operator who to call first.
4. Discovery kit: the 5-question intake, the Tier 1/2/3/Pause thresholds, the live document-ask script, and a templated Tier 1 brief. Mostly process plus templates, little code.
5. Document engine: S2-to-S3 stage trigger writes a document-request draft to pending_actions, arms a 48-hour chase scan, with a pg_cron backup that emails the operator independent of n8n. Plus the funder_submissions tracker (stips, cleared, offer_expiry). This plugs the biggest leak between verbal yes and funded file.
6. Small VPS (Hetzner or Contabo, 6 to 10 USD) running self-hosted n8n via Docker with restart=always, a UptimeRobot ping to Telegram, and the config.system_halt kill switch. Resolves the host question; a mini PC is cut.
7. Model routing and caps: Claude for briefs, note cleanup, and document drafts (a few dollars per month); DeepSeek only for anonymized non-PII formatting behind a payload-scrubbing node; hard billing caps with a fail-closed model_blocked flag. Resolves the DeepSeek question; DeepSeek is barred from PII.
8. Opt-out suppression: a hardcoded keyword check that writes opted_out directly before any model call, read as the first condition in every outbound workflow.
9. Qualification page: distress gate at field 3-4, progressive radio-bucket fields, ONE payback calculator (state-gated, off for CA until SB 362 clears), named non-pre-checked consent, deterministic n8n scoring and routing, and a plain-form fallback if the booking embed fails. Built now, but goes live only after gate 2.

## Sequencing (critical path)

- Week 1: items 1 to 8 above. Operator begins calling the warm book (EBR, with DNC and call-window checks) and closing by roughly day 3 to 5. In parallel, the operator and counsel work the compliance gates; that track is not blocked by the build.
- Week 2, gated by compliance sign-off: page live (item 9), then the UCC pipeline as a manual pilot (pull, filter to cleared states and the 60-100 day window, normalized dedupe, Apollo enrichment pre-filtered to 25 or fewer, static payoff estimator), feeding manual-dial cold outreach. SMS stays off.
- Phase 2: see defer list.

## Defer (Phase 2, with the trigger that promotes it)

- SMS channel. Trigger: consent copy approved and page consent records flowing, counsel confirms the language names text.
- Full inbound reply classifier (six-label model). Now: keep only the hardcoded opt-out guard and let the operator read 10-30 replies a day by hand. Trigger: inbound volume exceeds manual capacity.
- UCC pull automation on a schedule. Now: manual. Trigger: two clean manual runs.
- End-of-day report automation, local Ollama model, People Data Labs fallback, referral ask. Triggers: operator time pressure, model-spend pressure, Apollo shortfall, first funded deals (referral).

## Cut (with reason)

- Three-agent architecture. No reasoning load at 10-30 touches a day; deterministic n8n plus one Supabase query replaces it. Answers open question 1.
- Mini PC host. Single point of failure during the prime call window. Answers open question 2.
- DeepSeek for any PII or financial data. No US residency, no DPA; not worth saving about 2 USD. Answers open question 3.
- Ringless voicemail, aged-lead vendor spend, in-window LinkedIn, A/B testing infrastructure, the separate funding-readiness check, the 30-minute consultative call, automated Tier 3 drip.
- Engineering gold-plating for this volume: event-sourcing or CQRS, real-time subscriptions, normalized stip and consent tables, data-lake layers, embedding search and thread reconstruction on replies, LLM-as-judge, cross-provider shadow testing.

## Budget envelope (within 300 USD)

VPS 6-10, Supabase free, n8n self-hosted free, Cal.com free, page host free, Claude and OpenAI 3-10, DeepSeek 0-2, UCC vendor 50-80, Apollo 0-49, DNC subscription 0 to roughly 82 per area code beyond the first 5, wireless lookup small. Realistic total 70-180 USD. The swing costs are DNC per-area-code fees and a paid Apollo tier; both fit under the cap with buffer.

## Scope risks I am accepting

1. The warm book may be too small to hit the 90-day number alone, so the cold engine must clear compliance fast or revenue stalls. I accept this because the alternative (cold outreach before registration) is the larger risk.
2. Manual reply handling and manual UCC pulls cost operator time. I accept this to avoid building automation before the motion is validated.
3. Counsel turnaround on consent and SB 362 is outside the operator's control and is the true critical-path dependency for the cold engine. Start it day one.
