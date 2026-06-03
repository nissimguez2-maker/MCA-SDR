**Lane:** Intake, enrichment, and deduplication pipeline for all three lead sources: company-provided, CRM warm-book export, and UCC-1 self-sourced.

---

**Implementing the strategy:**

**Tactic 1 (UCC near-payoff targeting):** Build a deterministic payoff-date estimator, not a model. MCA terms cluster at 3, 4, 6, and 9 months. Filing date plus the funder's modal term gives a point estimate. Confidence interval is wide (20-plus days), but the 60-70 percent payoff window (targeting 60-100 day filings) is just a filter: `WHERE days_since_filing BETWEEN 60 AND 100`. No ML needed. Flag filings under 60 days as too early; flag above 120 days as likely paid or renewed. Revisit at 90 days if unfired.

**Tactic 3 (warmth scoring and routing):** The pipeline must write `monthly_revenue`, `time_in_business_months`, `stack_count`, and `funded_date` as structured columns into Supabase before the scoring formula runs. If enrichment fails to populate these fields, the lead gets a null-penalty score, not a silent zero. This is a data contract: scoring depends on the pipeline honoring the schema.

**Tactic 5 (daily queue):** The queue formula needs `clawback_window_end` and `active_deal_flag` per merchant. The pipeline must set these at ingest time, not at queue-build time, or the queue will occasionally pull a merchant mid-deal. Active deal check and clawback block must be idempotent flags on the Supabase row.

---

**Build spec:**

Three source flows, one canonical leads table.

**Source A: Company-provided leads.** Operator pastes CSV or forwards email. n8n webhook or scheduled poll picks it up. Parse: business name, owner name, phone, email, monthly revenue, TIB. Normalize phone to E.164. Write to `leads_raw` with `source = 'company'`. Skip enrichment if phone and email are both present and non-null; otherwise enrich.

**Source B: CRM warm-book export.** One-time or periodic CSV from the brokerage CRM. Same parse as Source A plus `funded_date`, `funded_amount`, `funder_name`. Derive `time_since_funding_days = today - funded_date`. Write to `leads_raw` with `source = 'warm_book'`. These have direct contact info; enrichment is a fill-in-the-blanks step only.

**Source C: UCC-1 self-sourced.** Pull from a sub-100 USD UCC vendor (LIEN Solutions UCC Search or Secretary of State bulk exports for TX, FL, NY, CA at roughly 50-80 USD per state pull). Parse fields: debtor business name, debtor address, secured party (funder) name, filing date, filing number. Derive `days_since_filing`. Filter immediately: `days_since_filing BETWEEN 60 AND 100` and `state IN (cleared_states_list)`. Reject records missing debtor business name or filing date. Write survivors to `leads_raw` with `source = 'ucc'`.

**Enrichment step (UCC and partial company leads only).** Apollo.io free tier: 50 credits per month on the free plan. At roughly 2 credits per contact lookup, that yields 25 enriched contacts per month. This is the hard ceiling. Prioritize UCC leads because company leads usually arrive with contact info. Apollo lookup key: business name plus state. Target fields: owner first name, owner last name, direct mobile, work email. Write result to `leads_enriched`. If Apollo returns null on direct mobile after one attempt, mark `enrich_status = 'failed'` and do not retry automatically. Flag for manual LinkedIn lookup (cross-lane: outreach agent handles this, not the pipeline).

Apollo free tier caps at 25 enriched UCC leads per month. At 10-30 leads per day (up to 30 days in a 60-100 day window), a targeted state-filtered UCC pull will exceed that fast. Resolution: upgrade Apollo to the Basic plan at 49 USD per month (1,200 credits, roughly 600 lookups), or use People Data Labs free tier (100 free API calls per month) as a fallback for overflow. Keep total enrichment spend under 60 USD per month to leave budget for UCC vendor fees.

**Deduplication.** Dedupe key: normalized phone (E.164) as primary, business name plus state as secondary. Steps in order: (1) exact phone match against `leads_canonical`, (2) fuzzy business name match (Postgres `pg_trgm` similarity above 0.85) scoped to same state. On match, merge into the existing row: update `source_list` (array), set `last_seen_at`, do not create a new row. On no match, insert. Dedupe also runs against two blocking lists: `active_pipeline` (any lead with a deal in S1-S5) and `clawback_block` (any funded merchant where `funded_date > today minus 90 days`). Any match against these lists sets `contact_block = true` and suppresses the lead from the queue without deleting it. This is the gate that prevents calling a merchant already in a live deal.

**Data quality gates before a lead enters the queue.** Required fields: business name not null, phone not null and valid E.164, `source` not null. Soft requirement: owner name, email, `monthly_revenue`. A lead missing hard requirements is written to `leads_quarantine` with a `fail_reason` column. Quarantined leads fire a Slack or email alert to the operator once per day (not per record). Soft-missing leads enter the queue with a reduced warmth score.

**Payoff-date estimation.** For UCC leads: `estimated_payoff_date = filing_date + funder_modal_term_days`. Funder term lookup: a small static table in Supabase mapping known MCA funder names (from UCC secured party field) to their modal advance term in days. Start with a default of 120 days (4 months) for unknown funders. Target the 60-70 percent payoff point at `filing_date + 0.65 * modal_term`. This yields the `payoff_confidence` field (high if funder is in the lookup table, low if using the default). Track actuals and update the lookup table monthly.

**Pipeline cadence.** UCC pull: manual, monthly, operator-triggered (given the volume). Company leads: on-receipt via n8n webhook. CRM export: weekly scheduled import. Enrichment: runs within 5 minutes of a new UCC or partial lead entering `leads_raw`, using an n8n workflow triggered by Supabase insert. Queue rebuild: nightly at 03:00 ET (before the operator's shift starts), plus on-demand trigger.

**Schema (core table: `leads_canonical`).** Key columns: `lead_id` (UUID), `business_name`, `owner_name`, `phone_e164`, `email`, `state`, `source` (array), `monthly_revenue`, `time_in_business_months`, `stack_count`, `funded_date`, `funder_name`, `filing_date`, `days_since_filing`, `estimated_payoff_pct`, `payoff_confidence`, `enrich_status`, `contact_block` (bool), `active_deal_flag` (bool), `clawback_window_end` (date), `warmth_score`, `queue_tier`, `created_at`, `updated_at`, `source_system`.

---

**Over-built or cut:**

- No Spark, no Delta Lake, no dbt, no Great Expectations. Postgres constraints plus a nightly n8n validation job that counts nulls on required fields and posts a summary is sufficient at 10-30 leads per day.
- No streaming pipeline. Batch ingest with a 5-minute enrichment trigger is fast enough.
- No ML payoff model. A static lookup table with a 120-day default is auditable, maintainable by one person, and correct enough for the 60-100 day filing filter.
- No separate bronze/silver/gold layers. One `leads_raw` staging table and one `leads_canonical` table, with `leads_quarantine` for failures. Three tables, not a lakehouse.
- No data catalog tooling. Column comments in Postgres are sufficient documentation at this size.

---

**Top risks:**

1. **Apollo free tier exhausted in week one.** A single targeted UCC state pull (TX alone) can return hundreds of records in the 60-100 day window. If the operator does not pre-filter aggressively by funder name and state before sending to Apollo, 50 free credits disappear immediately and enrichment stalls for the month. Mitigation: Apollo enrichment runs only after the UCC filter reduces the list to 25 or fewer records, or the operator upgrades to 49 USD per month Basic before the first pull.

2. **Contact-block flag not set before the queue runs.** If `active_deal_flag` or `clawback_window_end` is populated by a separate CRM sync that runs after the queue rebuild, the operator can be handed a call for a merchant already mid-deal. Mitigation: the queue-rebuild n8n workflow must join against the active pipeline table in real time, not rely on a pre-computed flag. The flag is a cache; the join is the gate.

3. **UCC debtor name does not match CRM or company-lead business name.** Fuzzy match at 0.85 similarity will miss LLCs registered as "ACME ENTERPRISES LLC" when the CRM has "Acme Enterprises." Missed deduplication means a warm-book merchant gets a cold UCC outreach, which signals disorganization. Mitigation: normalize to uppercase, strip legal suffixes (LLC, INC, CORP, CO) before deduplication, then apply `pg_trgm`. Document the normalization rules; review the quarantine table weekly for patterns.

---

**Constraint flags:**

- UCC pulls must be filtered to states where the operator has verified broker registration and commercial financing disclosure compliance before any outreach is triggered. The pipeline must enforce a `cleared_states` allowlist; do not let a record from an uncleared state reach the queue. This is a legal gate, not a data-quality gate.
- Apollo and People Data Labs free tiers require compliance with their data use terms. Enriched contact data (owner mobile) used for outreach against a UCC-sourced lead without page opt-in is the TCPA one-to-one consent unresolved conflict flagged in the strategy brief. The pipeline can enrich and store the data; whether it can be used for a cold call or text is a legal question that must be resolved before the pipeline feeds the outreach cadence. The pipeline should set a `consent_status` field and only allow the queue to surface UCC leads where `consent_status = 'cleared'` or `consent_status = 'pending_legal_review'` with a visible flag.
- DeepSeek access to owner phone and email fields raises the data residency concern flagged in plan.md. Until that question is resolved, enriched PII should not be passed to DeepSeek as prompt context. Supabase column-level encryption is worth enabling for `phone_e164` and `email` at setup, not as a retrofit.

---

**Cross-lane flags:**

- Manual LinkedIn lookup fallback for enrichment failures belongs to the outreach agent lane, not this pipeline. The pipeline sets `enrich_status = 'failed'`; the outreach agent decides whether to attempt manual enrichment or suppress the lead.
- The `cleared_states` allowlist and broker registration verification are legal/compliance lane. The pipeline enforces the list; it does not maintain it.
- TCPA consent resolution (B2B cell calls on UCC-sourced contacts without page opt-in) gates whether the enriched phone field can be used at all. This is an unresolved conflict from the strategy brief and must be resolved by the deal strategist or legal review before UCC outreach goes live.
- The funder modal-term lookup table needs real data from the brokerage's funded book and UCC observation over time. The growth hacker or deal strategist lane should provide initial funder term priors.

---

**Facts to verify:**

- Apollo.io free tier: exact credit count per month, credit cost per contact enrichment lookup (1 vs. 2 credits), and whether business name plus state is a supported lookup key without a LinkedIn URL.
- People Data Labs free API tier: monthly call limit, fields returned (direct mobile included or business line only), and data use terms for outreach.
- Postgres `pg_trgm` similarity threshold performance at 500-2,000 records: confirm it does not require an index rebuild or extension install on Supabase free tier.
- Sub-100 USD UCC vendor with bulk debtor export: confirm LIEN Solutions, CSC Global, or Secretary of State direct bulk download pricing for TX, FL, NY, CA. Verify export format includes secured party (funder) name as a parseable field, not a free-text blob.
- Supabase free tier row limits and whether column-level encryption (pgcrypto) is available without a paid plan.
- Modal MCA advance terms by top funder names (OnDeck, Kabbage, Credibly, Rapid Finance, Libertas, etc.) to seed the funder term lookup table.
