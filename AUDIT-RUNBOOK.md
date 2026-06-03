# MCA SDR Plan Audit: Self-Contained Runbook

This file is the whole audit. Claude Code: read this and `plan.md`, then execute the steps in order. You do not need any external repository; every lens is defined below. (Optional: if the agency-agents subagents happen to be installed, you may route each lens to its matching subagent. If not, adopt each lens yourself.)

## Rules for execution

- Work one numbered step at a time. After each step, write the output file or files, show what you produced, and wait for the operator to say continue before the next step.
- For each lens in a pass, fully adopt that specialist perspective and audit only its lane. Be specific and blunt. No generic praise.
- Keep each lens audit under 400 words and each reconciliation brief under 600 words.
- Write outputs to the paths given. Create folders as needed: `outputs/pass1`, `outputs/pass2`, `outputs/pass3`, `outputs/final`.
- No em dashes anywhere in any output. Operator preference.
- If web search is available, use it for the Pass 1 market lenses and the compliance lens. Otherwise, list claims to verify instead of asserting them.
- The two reconciliation briefs (Steps 2 and 4) are narrowing steps, not summaries. They cut. After each, invite the operator to edit the brief before continuing.

## Shared context (every lens uses this)

- Operator: a solo SDR at a US small-business funding / merchant cash advance (MCA) brokerage, on a 90-day commission run, working from Israel on US hours (about 17:00 to 02:00 Israel time, 10:00 to 19:00 US Eastern).
- Goal: maximize funded MCA deals. Judge everything by whether it produces more qualified conversations and more funded files.
- Hard constraints: 300 USD per month total tooling budget, not a dollar over; one person building and maintaining it outside working hours, so anything that breaks during work hours costs live selling time; must be always-on for events and scheduled briefs; must not become an overbuilt agent lab; outreach must be compliant and non-deceptive.
- Warming model: the operator's own outbound drives contacts to a single smart qualification page that self-qualifies and warms them; warm or booked leads rank to the top of a daily call queue; the page is a capture and consent surface downstream of outbound, not a traffic source.
- Proposed stack, open to challenge: Supabase as source of truth, n8n as a thin connector layer only, Hermes agents as the brain, DeepSeek as default model with Claude or OpenAI for high-judgment work, Cal.com or Calendly for booking, always-on host undecided (mini PC or small VPS).
- Full detail is in `plan.md`.

## The chain

Pass 1 Market, then reconcile to market-brief, then Pass 2 Strategy reads it, then reconcile to strategy-brief, then Pass 3 Implementation reads it, then scope ruling, then final reconcile to FINAL-PLAN.

---

## STEP 1: Pass 1, Market and opportunities

Read `plan.md` for context, then keep focus on the external market. Run these six lenses one at a time:

1. Trend Researcher: market size, segments, competitive landscape, where MCA demand concentrates.
2. Outbound Strategist: which merchant signals indicate funding need, the ideal customer profile, where warm prospects are found.
3. Investment Researcher: segment economics, which merchant profiles are fundable and profitable, risk.
4. Loan Officer Assistant: what the funding process actually requires, what makes a merchant approvable.
5. Reddit Community Builder: where merchants gather, the questions and pain points they voice, unmet needs.
6. Search Query Analyst: the search intent and language merchants use when seeking funding, demand signals.

Output per lens, written to `outputs/pass1/<lens>.md`:

- Lane: one line.
- Market reality: what is true in this lane now.
- What the market needs: specific unmet or underserved needs.
- Opportunities to seize: max 4, ranked, concrete for a solo operator.
- Hard truths and risks: max 3.
- Facts to verify: exact external claims to confirm.
- Cross-lane flags.

## STEP 2: Reconcile Pass 1 into market-brief.md

No persona. Read every file in `outputs/pass1`. Narrow, do not merely summarize: dedupe, rank by impact on funded deals under the constraints, recommend the top opportunities to pursue and park the rest. Write `market-brief.md`:

- Top opportunities to pursue: max 5, ranked, each with the lanes behind it and a one-line reason.
- Parked opportunities: real but not now, with the reason.
- Key market needs to address.
- Conflicts: where lenses disagreed, one line each.
- Facts to verify: consolidated.

Then invite the operator to edit `market-brief.md` before Step 3.

## STEP 3: Pass 2, Strategy and tactics

Read `market-brief.md` and `plan.md`. Run these six lenses one at a time:

1. Deal Strategist: positioning and win strategy for the chosen opportunities.
2. Sales Outreach: the cold-to-warm cadences, openers, multi-touch sequences.
3. Discovery Coach: call structure and the qualifying questions behind tiering.
4. Behavioral Nudge Engine: the warming-page strategy, progressive qualification, completion and booking lift.
5. Growth Hacker: the outbound-to-page funnel design and lean experiments to test it.
6. Pipeline Analyst: prioritization strategy, deal velocity, where to focus limited solo time.

Output per lens, written to `outputs/pass2/<lens>.md`:

- Lane: one line.
- Strategy: short paragraph, how to seize the top opportunities in this lane.
- Tactics: max 5, ranked, concrete.
- What to avoid: tactics that waste a solo operator's time or budget here.
- Over-built or cut: what in the original plan, in this lane, the strategy makes unnecessary.
- Top risks: max 3.
- Cross-lane flags.
- Facts to verify.

## STEP 4: Reconcile Pass 2 into strategy-brief.md

No persona. Read every file in `outputs/pass2`. Narrow: pick the winning strategy and the tactics that implement it, park the alternatives. Write `strategy-brief.md`:

- Chosen strategy: short paragraph.
- Key tactics: max 6, ranked, each tagged with the lanes behind it.
- Parked or alternative tactics: with the reason.
- Constraints to respect: budget, compliance exposure, solo-operator limits.
- Conflicts: unresolved disagreements for later steps.
- Facts to verify: consolidated.

Then invite the operator to edit `strategy-brief.md` before Step 5.

## STEP 5: Pass 3, Implementation

Read `strategy-brief.md` and `plan.md`. Run these six lenses one at a time:

1. Backend Architect: Supabase schema, tables, events, source-of-truth design.
2. Data Engineer: the intake, enrich, and dedupe pipeline for sourced and company leads.
3. Email Intelligence Engineer: parsing inbound replies into structured, reasoning-ready records.
4. Automation Governance Architect: the n8n and agent layer, failure surface, what runs when, permissioning.
5. Autonomous Optimization Architect: model routing, token tracking, the 300 USD cap as a hard guardrail.
6. Legal Compliance Checker: TCPA, DNC, state MCA disclosure, consent capture. List claims to verify.

Output per lens, written to `outputs/pass3/<lens>.md`:

- Lane: one line.
- Implementing the strategy: for each relevant tactic, the leanest concrete build, or a pushback saying it is not worth building and why.
- Build spec: concrete. Name what runs, when, and where.
- Over-built or cut: required.
- Top risks: max 3, especially anything that breaks during work hours.
- Constraint flags: legal requirements or budget risks that gate the build.
- Cross-lane flags.
- Facts to verify.

## STEP 6: Scope ruling

Adopt the Senior Project Manager as scope authority. Read `strategy-brief.md` and every file in `outputs/pass3`. Rule on what to build now, what to defer, and what to cut, to reach the smallest version that still closes deals under the 300 USD per month and one-operator constraints. Write `outputs/final/scope-ruling.md`.

## STEP 7: Final reconcile into FINAL-PLAN.md

No persona. Read `plan.md`, `market-brief.md`, `strategy-brief.md`, every file in `outputs/pass3`, and `outputs/final/scope-ruling.md`. Apply these rules:

- Dedupe and merge overlapping items.
- Resolve conflicts explicitly: state each in one line, name the lanes, resolve toward the leaner option that still closes deals.
- Scope ruling is decisive on cut or defer, unless a compliance blocker or a direct deal-closing function overrides it.
- Compliance items are blocking and go at the top.
- Keep the through-line: every Build Week 1 item must trace to a chosen tactic, and every tactic to a market opportunity. Cut anything that does not trace back.
- Respect the constraints throughout.

Write `FINAL-PLAN.md`:

- Executive verdict: 3 to 4 sentences, including the single biggest change required.
- Market opportunity: one short paragraph.
- Chosen strategy: one short paragraph.
- Compliance blockers: resolve before any outreach.
- Build Week 1: each item with a one-line reason, the tactic it serves, and the lanes behind it.
- Phase 2 backlog: ordered by deal impact.
- Cut: each with a one-line reason.
- Unresolved conflicts for the operator to decide: clear either/or choices.
- Facts to verify: consolidated across all passes.
