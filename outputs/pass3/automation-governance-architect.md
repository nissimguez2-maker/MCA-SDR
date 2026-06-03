# Pass 3 Audit: Automation Governance Architect

**Lane:** n8n and agent layer, failure surface, what runs when, permissioning, resilience, host.

---

## Implementing the strategy

**Tactic 2: Gating page + consent capture**

Build one webhook in n8n that receives the form POST from the qualification page. Input validation step: reject submissions missing phone, revenue, or time-in-business fields; write a rejected-submission row to Supabase for audit. Consent flag must be a boolean field in the payload; if absent or false, write the record to `leads` with `consent_sms = false` and suppress any SMS step downstream. Do not rely on the page front-end to enforce this; validate it server-side in n8n.

**Tactic 3: Deterministic scoring and routing**

APPROVE. Hard-coded point table lives in a single n8n Code node, not in a model call. The scoring function reads five fields: form_complete, real_phone_flag, callback_requested, booking_created, time_on_page_bucket. Output: score integer, tier label, and route destination. Score 80-plus writes `route = booking_link`; 50-79 writes `route = sdr_queue`; under 50 writes `route = nurture`. Exactly one Supabase upsert on `leads.score` and `leads.tier`. No agent, no model API call. This is the canonical scoring path; no parallel scoring logic anywhere else.

Workflow name: `PROD-CRM-LeadIntake-ScoreRoute-v1.0`

**Tactic 5: Pre-shift command brief + call-window enforcement**

Two workflows.

`PROD-OPS-ShiftPrep-CommandBrief-v1.0`: schedule trigger at 09:30 ET daily (Monday through Friday). Queries Supabase for Tier 1 leads ranked by composite score, today's bookings from Cal.com API, and document-chase items due today. Formats a structured brief (Do Now, Do Next, Hot Alerts). Sends via email (SMTP or Postmark) and optionally Telegram. No model call needed at this volume: brief is a templated SQL result. APPROVE.

`PROD-OPS-CallWindow-LocalTimeCheck-v1.0`: called as a sub-workflow before any outbound action node writes or sends anything. Inputs: merchant state (from Supabase), proposed send time (UTC). Checks merchant-local hour. If outside 08:00-21:00, writes to a `deferred_actions` table and exits without sending. This is the only TCPA call-window gate; it must be called by every outbound workflow, not embedded ad-hoc. APPROVE.

**Tactic 6: Document chase as stage-transition trigger**

`PROD-CRM-DocChase-StageTransition-v1.0`: triggered by Supabase webhook on `leads.stage` changing to `S3`. On trigger: write `s3_entered_at` timestamp, generate document-request email draft (templated, no model call needed for standard request), set `doc_chase_due_at = now + 48h`. A separate scheduled workflow (`PROD-CRM-DocChase-FiftyHourScan-v1.0`) runs every two hours during 09:00-19:00 ET and queries for records where `doc_chase_due_at < now` and `stage = S3` and `docs_received = false`. It queues a chase action for operator review, not auto-send. The operator approves each chase email in a lightweight approval table (`pending_actions`). Auto-send is blocked until the operator checks a row in Supabase or a simple approval webhook. PARTIAL AUTOMATION ONLY: generate and queue, human sends.

**Form webhook to score to route to CRM write to nudge: full path**

1. Page form POST hits n8n webhook (`PROD-CRM-LeadIntake-ScoreRoute-v1.0`).
2. Input validation: required fields, consent flag, phone format.
3. Dedup check: query Supabase for existing `phone` match; if exists, update record, skip insert.
4. Score computation (Code node, hard-coded table).
5. Upsert `leads` row with score, tier, route, consent fields, and `intake_at` timestamp.
6. If score 80-plus and `booking_link` route: send booking-link email via SMTP (auto-send, low risk). If 50-79: insert into SDR queue, no outbound. If under 50: set `nurture_eligible = true`, no outbound.
7. If consent_sms = true and score 80-plus: queue a single SMS via Twilio (but flag to `pending_actions` for operator confirmation before first send per lead; auto-send only on confirmed-consent repeat contacts). PARTIAL AUTOMATION ONLY for SMS.
8. Log row: workflow name, version, lead_id, score, route, timestamp, success/fail.

**Weekly UCC pull to enrich to dedupe to queue**

`PROD-LEADS-UCCPull-EnrichDedupeQueue-v1.0`: manual trigger only for 90-day run. Reason: UCC vendor API reliability and cost are unverified; operator must validate the pull format and enrichment match rate before scheduling. Weekly schedule can be added in week 3 after one manual run is validated. Operator runs this outside work hours. It pulls the CSV or API result, enriches phone via Apollo free tier, dedupes against existing `leads` on phone and business name, writes net-new rows to a `ucc_queue` table with status `pending_review`. Operator reviews before any outreach is created. APPROVE AS PILOT.

**Inbound reply parser with opt-out suppression**

`PROD-OPS-InboundReply-ParseOptOut-v1.0`: triggered on inbound email webhook (Postmark inbound or equivalent). Checks body for opt-out keywords (STOP, unsubscribe, remove me, do not contact). If matched: set `leads.opt_out = true`, `leads.consent_sms = false`, write timestamp, and block all outbound workflows for that lead_id. Auto-suppression of opt-outs is the one place auto-write is fully approved without operator confirmation, because delay creates compliance exposure. APPROVE.

**End-of-day report**

`PROD-OPS-EndOfDay-Report-v1.0`: schedule trigger at 19:15 ET Monday through Friday. SQL aggregation from Supabase: calls logged today, S2-to-S3 conversions, S3-to-S4 completions, doc-chase items still open, opt-outs, queue depth by tier. Sends via email. No model call. APPROVE.

---

## Build spec

**Workflows (7 total):**

| Name | Trigger | Approvals | Auto-send? |
|---|---|---|---|
| PROD-CRM-LeadIntake-ScoreRoute-v1.0 | Webhook (form POST) | None for routing; SMS needs operator confirm | Booking-link email: yes. SMS: no. |
| PROD-OPS-ShiftPrep-CommandBrief-v1.0 | Schedule 09:30 ET M-F | None | Yes (to operator only) |
| PROD-OPS-CallWindow-LocalTimeCheck-v1.0 | Sub-workflow call | None | Blocks send if outside window |
| PROD-CRM-DocChase-StageTransition-v1.0 | Supabase webhook (stage change) | Operator approves each chase via pending_actions | No outbound auto-send |
| PROD-CRM-DocChase-FiftyHourScan-v1.0 | Schedule every 2h, 09:00-19:00 ET | Same pending_actions queue | No |
| PROD-LEADS-UCCPull-EnrichDedupeQueue-v1.0 | Manual (promote to schedule in week 3) | Operator reviews ucc_queue before outreach | No |
| PROD-OPS-InboundReply-ParseOptOut-v1.0 | Inbound email webhook | None (auto-suppress is required) | Suppression write: yes |
| PROD-OPS-EndOfDay-Report-v1.0 | Schedule 19:15 ET M-F | None | Yes (to operator only) |

**Approval mechanism:** a `pending_actions` table in Supabase with columns `id`, `workflow`, `lead_id`, `action_type`, `payload`, `status` (pending/approved/rejected), `created_at`, `actioned_at`. n8n polls this table on a 5-minute schedule during work hours and executes approved rows. The operator flips `status` in Supabase Table Editor or a minimal linked form. No custom UI needed in week one.

**Host recommendation: small VPS (not mini PC).**

Rationale: operator is in Israel; work hours are 10:00-19:00 ET; a home mini PC is dark and offline during that window if power, ISP, or the machine fails, with no remote recovery path during a live selling day. A 6 USD/month VPS (Hetzner CX22 or Contabo S) stays up in a data center with an SLA. n8n self-hosted on that VPS costs nothing extra. Risk of host downtime is lower and recovery is SSH-accessible from anywhere.

**Resilience mechanisms:**

- All workflows have an explicit error branch that writes to a `workflow_errors` table (workflow name, version, entity_id, error_class, short note, timestamp) and sends a Telegram or email alert to the operator.
- Retries: webhook workflows retry twice with 30-second backoff before hitting the error branch. Scheduled workflows do not retry; the next scheduled run picks up missed items via the state already in Supabase (idempotent by design).
- Dead-letter: failed lead intakes write to `intake_errors` with the full payload so no submission is silently lost.
- Kill switch: a single `system_halt` boolean in a Supabase `config` table. Every outbound action node checks this flag before executing. Setting it to true stops all outbound without touching the VPS.
- Booking embed failure: the plain-form fallback (a static HTML form posting to the same n8n webhook) must be linked directly from the booking section of the page. No JavaScript dependency.
- n8n free-tier execution cap: n8n Cloud free tier caps at 5,000 executions per month. At 10-30 form submissions per day plus 8 scheduled workflows running daily, peak consumption is roughly 1,500 executions per month. This is within the free tier. If volume rises, migrate to n8n self-hosted on the VPS (the recommended host already), which has no execution cap.

---

## Over-built or cut

**Three-agent architecture (Warm Lead Engine, Sales Intelligence Agent, Operator/Secretary Agent): CUT.**

These collapse to: 7 deterministic n8n workflows plus one composite Supabase query for the ranked queue. There is no reasoning task in the 0-30-lead-per-day range that requires an agent. Agents add model API cost, latency, and failure modes. The brief, the routing, the document request, and the report are all templated SQL results. Introduce one Hermes agent only when after-call note cleanup volume justifies it (roughly 20-plus calls per day with structured notes) or when scoring rules need iterative refinement beyond a Code node.

**Separate scoring agent: CUT.** Hard-coded point table in one Code node. Reviewable, version-controlled, zero API cost.

**Automated Tier 3 nurture sequences: CUT (as per strategy).** Non-responders get a single 90-day re-ping queued manually. No drip workflow.

**Any LLM call inside a scheduled workflow: DEFERRED.** All scheduled workflows use templated SQL output. Model calls are reserved for operator-initiated tasks (brief refinement, note cleanup) and must be triggered manually to keep API costs predictable.

---

## Top risks

**Risk 1: VPS or n8n process goes down during 10:00-19:00 ET with no one to restart it.**
Mitigation: set up a process monitor (systemd service + auto-restart) and an uptime check (UptimeRobot free tier) that alerts Telegram within 1 minute of the n8n process dying. The operator can SSH-restart from a phone if needed. Without this, a webhook drop during work hours means missed lead intakes with no recovery signal.

**Risk 2: Supabase webhook for stage transitions fires duplicate events, causing double document-chase emails.**
Mitigation: the stage-transition workflow must check `s3_entered_at IS NULL` before writing it; if already set, exit immediately. This is the idempotency guard. Without it, any retry or duplicate webhook fires a second document request to the same merchant, which is unprofessional and potentially damaging.

**Risk 3: Cal.com booking webhook drops or the embed fails to load, and no booking is recorded in Supabase, so the lead sits unranked.**
Mitigation: the plain-form fallback captures the booking intent and posts to n8n directly. Additionally, the command brief query should surface any leads with `route = booking_link` and `booking_created = false` older than 4 hours as a manual follow-up item, so the operator catches the gap at shift start even if the webhook was silent.

---

## Constraint flags

- **TCPA consent gates every SMS action.** The inbound-reply opt-out suppression is non-negotiable and must auto-execute without operator approval. All other outbound SMS requires `consent_sms = true` confirmed at the page. Any workflow that could send an SMS must check this flag as the first condition, before any other logic. This is a hard gate, not a best practice.
- **n8n free-tier execution cap (5,000/month)** must be monitored. If the VPS host is used, self-host n8n to eliminate the cap entirely. Given the VPS recommendation, self-hosted n8n is the correct default.
- **Budget:** the VPS at 6-10 USD/month fits within the 300 USD budget. Postmark or SMTP for transactional email adds 0-10 USD/month on a free tier. Total automation infrastructure cost is under 20 USD/month, leaving budget for model API and enrichment.
- **DeepSeek access to PII and bank statements:** any workflow that passes bank statement data or owner SSN/EIN to a model must use Claude or OpenAI (not DeepSeek), given DeepSeek's data-residency uncertainty for US business PII. Flag to the data architect lane. Deterministic n8n workflows do not call any model, so this constraint does not affect the 7 workflows above; it applies only to future note-cleanup or brief-enrichment workflows.

---

## Cross-lane flags

- **Supabase schema design** (schema/data architect lane): the `pending_actions` table, `workflow_errors` table, `intake_errors` table, `config` table (with `system_halt`), and the consent and opt-out fields on `leads` must be defined and documented before any workflow goes to production. This audit assumes they exist; they are not built here.
- **Qualification page and booking embed** (front-end/UX lane): the plain-form fallback must POST to the exact n8n webhook URL and include all required fields. The page team must confirm this before the lead-intake workflow is finalized.
- **TCPA consent copy** (compliance lane): the consent checkbox label and the page disclosure text require attorney review before any outreach begins. The workflow can be built in parallel, but outbound must not fire until that review is complete.
- **Cal.com webhook reliability and free-tier limits** (integration lane): the booking-to-Supabase write path depends on Cal.com webhooks firing reliably. If Cal.com free tier does not support outgoing webhooks, the queue-ranking formula cannot auto-promote booked leads. This must be verified before the command-brief workflow is finalized.

---

## Facts to verify

1. n8n Cloud free-tier execution limit (currently documented as 5,000/month): confirm this applies per workspace and whether scheduled sub-workflow calls count separately. If self-hosting on VPS, confirm n8n process memory requirements for this workflow count (7 workflows, low concurrency).
2. Cal.com free-tier outgoing webhook support: does the free plan fire a webhook on new booking creation? If not, the only alternative is polling the Cal.com API on a schedule, which adds latency to the command brief.
3. Supabase webhook (database webhooks via pg_net or Realtime): confirm whether a `BEFORE UPDATE` trigger on `leads.stage` can reliably fire an n8n webhook, or whether a Supabase Edge Function intermediary is required. The reliability of this path determines the doc-chase trigger architecture.
4. Postmark or equivalent inbound email webhook: confirm that a free or low-cost inbound parsing service can forward emails to an n8n webhook for opt-out parsing, and that forwarding latency is under 5 minutes (acceptable for opt-out compliance).
5. VPS provider choice: confirm Hetzner CX22 or equivalent is available in a US-East region (for Supabase co-location) and that the 6-10 USD/month tier provides sufficient uptime SLA for the operator's risk tolerance.
