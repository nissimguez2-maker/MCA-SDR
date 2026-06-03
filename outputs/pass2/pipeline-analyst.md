# Pass 2 Audit: Pipeline Analyst Lane

**Lane:** Prioritization strategy, deal velocity, and daily focus allocation for a solo operator.

## Strategy

The 20x EV gap between segment tiers is not a marketing insight; it is a time-allocation mandate. Every call placed on a high-risk, low-tenure, stacked merchant is a call not placed on a 30-50k USD/month, clean-file merchant with an 18-plus-month term. A solo operator with a 90-day commission window and a 60-90 day clawback exposure cannot recover from two wrong calls per day compounding for six weeks. The queue-ranking formula must encode this gap at the field level in Supabase, not in the operator's head each morning. Pipeline stage definitions must reflect where MCA deals actually stall (document submission, not verbal approval), and the daily work-mode calendar must protect the prime US call window from admin bleed.

## Tactics

**1. Queue-ranking formula (composite score, 0-100)**

Score each lead as: `(Warmth x 0.40) + (EV Tier x 0.35) + (Clawback Survivability x 0.15) + (Recency Penalty x 0.10)`.

- Warmth (0-10): booked call = 10, page completed with real phone = 8, re-engagement reply = 7, funded book re-contact = 6, warm aged lead = 4, cold = 1.
- EV Tier (0-10): monthly revenue 30k+ and 18+ month term and no stack = 10; 15-30k or 12-18 month = 6; sub-15k or under 12 months or stacked = 2.
- Clawback Survivability (0-10): funded-date field in Supabase; if prior deal is inside the 90-day clawback window, subtract 4 points from total score regardless of other signals. A default in week 8 of a 90-day run maps to roughly minus 576 USD in recovered commission (2 x 288 USD) against a 6,160 USD sweet-spot target.
- Recency Penalty (0-10): 10 = follow-up due today or tomorrow; score decays linearly to 5 at 7 days, 2 at 14 days, 0 at 30-plus days. This surfaces time-sensitive follow-ups without manual sorting.

Tier 1 = score 70+, call today. Tier 2 = 40-69, call or email after Tier 1. Tier 3 = below 40, nurture. Required Supabase fields: `monthly_revenue`, `time_in_business_months`, `stack_count`, `funded_date`, `offer_term_months`, `last_contact_date`, `warmth_signal`.

**2. MCA pipeline stage definitions with exit criteria and stall benchmarks**

| Stage | Exit Criteria | Median Days | Stall Flag |
|---|---|---|---|
| S1 Contacted | Verbal interest confirmed, business name and phone verified | 0-1 | 2+ days = re-dial or drop |
| S2 Qualified | Revenue and TIB on file, no fatal stip (default, < 6 months TIB, > 2 stacks) | 0-2 | 3+ days = re-qualify or reclassify |
| S3 Docs Requested | Bank statement request sent, docrequest timestamp logged | same day as S2 exit | 1 day = immediate chase |
| S4 Docs Received | 3 months bank statements in hand, clean (not expired, not partial) | 1-3 days post-request | 48 hours = document engine chase (Opp. 3) |
| S5 Submitted | File sent to funder, submission timestamp logged | same day as S4 exit | 4+ hours = follow up with funder |
| S6 Approved | Offer received, term and factor rate confirmed | 1-2 days | 24 hours no offer = second funder |
| S7 Funded | Wire confirmed, funded-date written to Supabase | 24-72 hours from approval | 48+ hours = call funder ops |

Partial bank statements are the single largest stall point (S3-S4). Every hour between S2 and S4 is a funded-deal risk. The document engine (Opportunity 3) does not run separately from this stage model; it is the mechanical rule set for S3-to-S4 transitions.

**3. Velocity leading indicators (watch these, not lagging revenue)**

- S1-to-S2 same-day conversion rate: target 60%+ on Tier 1 calls. Below 40% means qualification criteria or call openers are failing.
- S3-to-S4 48-hour completion rate: target 70%+. Below 50% means the document request template or the chase cadence has a gap.
- S4-to-S7 calendar days: target under 5 days on clean files. Track by funder. A funder consistently at 7+ days needs a second-funder default.
- Days-since-last-stage-change: flag any deal stalled at the same stage for more than 1.5x its benchmark. Stalled deals are dying deals; in MCA they do not recover without a human intervention in the same business day.

**4. Daily work-mode calendar (US 10:00-19:00 ET)**

| Block | Mode | Duration | Rule |
|---|---|---|---|
| 10:00-10:20 ET | Admin | 20 min | Read command brief. Confirm Tier 1 queue. No new research, no CRM cleanup. |
| 10:20-14:00 ET | Call | 3h 40min | Call Tier 1 only. Log rough notes after each call. No email drafting. |
| 14:00-15:00 ET | Follow-up | 1 hour | Chase docs on S3-S4 deals, send no-answer voicemail follow-ups, approve outbound emails. |
| 15:00-17:30 ET | Call | 2h 30min | Second call block: Tier 1 no-answers, Tier 2 warm. |
| 17:30-18:30 ET | Admin | 1 hour | After-call notes to CRM, document submissions, funder follow-ups, queue prep for tomorrow. |
| 18:30-19:00 ET | Meeting | 30 min | End-of-day review: stats, open loops, stall alerts, tomorrow's ranked queue confirmed. |

The Meeting mode is a hard stop for booked inbound calls or funder calls, inserted into the calendar by the operator or agent whenever scheduled. Call mode is protected; no admin task enters it.

**5. Tie the document engine to pipeline velocity, not to separate follow-up logic**

The post-call document request draft (Opportunity 3) must fire as a pipeline-stage transition action, not as a reminder. The trigger is: S2 exit confirmed. The n8n workflow writes S3 timestamp, generates the document request draft (bank statements, voided check, application), and sets a 48-hour chase event. If S4 is not reached within 48 hours, the chase fires automatically. This makes document velocity a measured metric (S3-to-S4 completion rate) rather than a discipline question. If that rate falls below 50%, the cause is diagnosable: wrong contact, wrong request format, or merchant went cold post-verbal.

## What to avoid

- **Approvability as the sort key.** Approving a 10k USD/month, 6-month advance sounds productive. The EV is approximately 288 USD before a likely clawback. Sorting by highest approval odds without tenure and revenue filters fills the day with sub-300 USD bets.
- **Pipeline inflation through stage drift.** If S2 exit is not gated on verified revenue and TIB, deals accumulate at S2 with no exit criteria, coverage looks healthy, and funded rate collapses. Stage-count is a vanity metric here; funded-in-30-days is the only real pipeline health number.
- **Mode-switching during Call blocks.** Each CRM update, email draft, or Slack check during the call window costs re-engagement time and compresses the number of Tier 1 dials. A solo operator with a 3h 40min call block who blends admin into it effectively has a 2-hour call block.
- **Recency bias in queue ordering.** The most recently added lead is not necessarily the highest-EV lead. The ranking formula must override the default CRM sort on every shift start.

## Over-built or cut

- The three-agent architecture described in plan.md section 7 bundles warmth scoring (Warm Lead Engine) and lead ranking (Sales Intelligence Agent) as separate agents. For this lane, they are a single scored output: one Supabase query joining warmth signals to EV tier to clawback window, run once before shift start. Two agents doing sequential work on the same data is latency with no added judgment.
- The "nurture" bucket in the tiering system does not need an automated workflow in the first 90 days. Tier 3 leads that do not become Tier 2 within 14 days should be exported to a follow-up list and called manually in the last 30 minutes of Follow-up blocks, not agent-managed. Building nurture automation before the core S1-S7 pipeline is validated wastes the build budget.

## Top risks

1. **Clawback concentration risk in week 6-9.** If the operator funds 3-4 deals in weeks 1-3 from high-risk merchants (score below 40 on EV Tier), and one defaults in week 7-8, the clawback wipes approximately 576-864 USD of commission (2-3 deals x 288 USD) against a 90-day target. The queue formula's EV Tier weight of 0.35 must be non-negotiable; it cannot be overridden for volume reasons mid-run.

2. **S3-to-S4 gap kills velocity silently.** A verbal yes that stalls on document submission does not appear as a loss in most CRM setups; the deal stays in S3 looking "in progress." Without a hard 48-hour chase rule and a measured S3-to-S4 rate, the operator will reach week 6 with a pipeline full of S3 stalls and no funded deals to show. This is the most common MCA SDR failure mode.

3. **Call window compression from Israel timezone drift.** The 10:00-19:00 ET window is already shorter than a US-based operator's available day. Any admin task that bleeds into the morning call block (10:00-14:00 ET) directly removes dials from Tier 1 leads, who have the highest funded probability. The command brief must be generated and ready before shift start, not during it.

## Cross-lane flags

- **Qualification page (Opportunity 2, cross-lane to UX/tech):** The warmth score inputs that feed the queue formula require consistent field output from the qualification page: specifically, the `monthly_revenue`, `time_in_business_months`, and `stack_count` fields. If the page does not capture these in a structured format, EV Tier scoring degrades to manual entry.
- **Document engine (Opportunity 3, cross-lane to automation/tech):** The S3-to-S4 trigger must be a pipeline stage event in Supabase, not a separate Follow-up workflow. The automation lane needs to build this as a stage-transition hook, not a time-based reminder.
- **UCC-1 self-sourcing (Opportunity 4, cross-lane to sourcing):** UCC-sourced leads enter the queue at warmth = 4 (warm aged, not verified intent). They must not consume Tier 1 Call time until they have received outreach and shown engagement. The sourcing lane should confirm the warmth-signal field UCC leads receive on import.
- **Compliance (plan.md section 11):** The funded-date and merchant PII fields in Supabase trigger state disclosure and consent requirements. The compliance lane (if separate) should confirm which fields require encryption or access controls before the database schema is finalized.

## Facts to verify

- Clawback window by funder: the audit uses 60-90 days; confirm whether any top funders (e.g., Libertas, Kapitus, BFS Capital, Greenbox) enforce shorter windows or deal-specific terms in their ISO agreements.
- Commission structure: the 288 USD / 2,520 USD / 6,160 USD EV figures are asserted in the operator context. Confirm the basis points or percentage per funded deal and whether ISO agreements cap total commission per deal or per period, as this directly changes the EV Tier weights.
- S3-to-S4 industry benchmark: the 48-hour document completion target is derived from the "clean file funds in 24-72 hours" fact. Verify whether this is funder-to-funder or specific to preferred funders in this brokerage's ISO network.
- Supabase field-level encryption options (relevant to `monthly_revenue`, `ssn_last4`, `bank_statement_url`): confirm whether row-level security alone meets the PII handling requirement or whether column-level encryption is needed given DeepSeek model access to these records.
