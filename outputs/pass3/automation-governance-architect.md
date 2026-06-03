# Pass 3 Audit: Automation Governance Architect

**Lane:** n8n and agent layer, failure surface, what runs when, permissioning, host, resilience.

---

## Implementing the strategy

**Tactic 2: Gating page + consent capture**
APPROVE. One n8n webhook receives the form POST. Input validation (step 2 of every workflow standard): reject payloads missing phone, monthly revenue, or time-in-business; write the rejected payload to `intake_errors` for manual review. TCPA consent checkbox must arrive as a boolean field; if absent or false, the record is written with `consent_sms = false` and every downstream SMS node is suppressed by checking that field, not by relying on the page front-end. The Cal.com booking embed has a documented failure mode: if the iframe does not load, the lead sees a broken page and is never captured. The page must render a static plain-form fallback (no JavaScript dependency) that POSTs to the identical n8n webhook URL. The n8n workflow must handle both paths: payload with a booking ID and payload without one. This is a page-build precondition; flag to the front-end lane.

**Tactic 3: Deterministic scoring and routing in n8n**
APPROVE. Score lives in a single Code node: five hard-coded signal weights (form completed, real phone flag, callback requested, booking created, time-on-page bucket). No model call, no external HTTP. Score 80-plus writes `tier=1, route=booking_link`; 50-79 writes `tier=2, route=sdr_queue`; under 50 writes `tier=3, route=nurture`. One Supabase upsert. Dedup check runs first: query `leads` on phone before insert; if exists, update and skip insert. This is the only canonical scoring path. No parallel scoring logic anywhere else in the system.

**Tactic 5: Pre-shift ranked queue and call-window enforcement**
APPROVE. One scheduled workflow at 09:30 ET (14:30 Israel) runs a single parameterized Supabase query: SELECT leads ORDER BY composite score DESC WHERE tier IN (1,2) AND next_action_due <= today AND merchant_local_hour BETWEEN 8 AND 21. The merchant-local-hour filter is the automatic 08:00-21:00 gate for the Israel-timezone risk. It lives in SQL, not in operator judgment. The query result drives a Markdown command brief (template node, no model call at this volume). Delivered via email plus optionally Telegram. Any lead with `route=booking_link` and `booking_created=false` older than 4 hours appears as a manual follow-up item in the brief, catching silent booking-webhook drops.

A sub-workflow (`PROD-OPS-CallWindow-LocalTimeCheck-v1.0`) must be called by every outbound action node before it sends. It reads the merchant's stored state and a timezone offset from Supabase, computes merchant local time, and if outside the window, writes the action to `deferred_actions` and exits without sending. This sub-workflow is not optional; every outbound workflow must call it.

**Tactic 6: Document chase as stage-transition trigger**
PARTIAL AUTOMATION ONLY. On `leads.stage` changing to S3, a Supabase row-change webhook fires n8n. The workflow: (a) checks `s3_entered_at IS NULL` as idempotency guard; (b) writes `s3_entered_at = NOW()`; (c) generates document-request email draft from a template node; (d) sets `doc_chase_due = NOW() + 48h`; (e) writes the draft to `pending_actions` with `status=pending`. A separate scheduled workflow runs every 2 hours during 10:00-19:00 ET and queries for S3 records where `doc_chase_due < NOW()` and `docs_received = false`. It queues a chase email to `pending_actions`. The operator approves each row via the Supabase Table Editor (flip `status=approved`). A 5-minute polling workflow (`PROD-CRM-PendingActions-Fire-v1.0`) checks for approved rows and executes the send. Document-chase emails never auto-send.

**UCC weekly pull to enrich to dedupe to queue**
APPROVE AS PILOT. Manual trigger for the first 3 weeks. Reason: UCC vendor API format and Apollo enrichment match rate are unverified at this brokerage. On manual run: pull CSV or API response, dedupe against `leads` on phone and business name, enrich owner phone via Apollo free-tier HTTP call, score raw records as tier-2 or tier-3 based on days-since-filing and estimated payoff percentage, write net-new rows to `ucc_queue` with `status=pending_review`. No outreach fires. Operator reviews and promotes to `leads` manually. After 2 validated runs, promote to Saturday 06:00 ET schedule.

**Inbound reply parser with opt-out suppression**
APPROVE, auto-execute without operator approval. On inbound email webhook (Postmark inbound parse or Gmail push), n8n checks the body for opt-out keywords (STOP, unsubscribe, remove me, do not contact) via a contains check, not a model. On match: write `leads.opted_out = true`, `leads.consent_sms = false`, `opted_out_at = NOW()`. Alert the operator via email. This is the one auto-write approved without a human checkpoint because delay creates compliance exposure. Every outbound workflow reads `opted_out` as its first condition, before any template or send node.

**End-of-day report**
APPROVE. Scheduled at 19:15 ET Monday through Friday. SQL aggregation: calls logged today, stage transitions, doc-chase items open, queue depth by tier, opt-outs. Template node renders it. Email delivery. No model call.

**After-call note cleanup**
PARTIAL AUTOMATION ONLY. Operator pastes rough notes into a manual-trigger workflow. One synchronous DeepSeek call produces a structured CRM draft. Draft is written to `pending_actions` with `type=crm_note`. Operator reviews and saves or discards. No auto-write to `leads` or `deals`. This is the only model call in the production system.

**Three-agent architecture**
REJECT. All three proposed agents (Warm Lead Engine, Sales Intelligence, Operator/Secretary) collapse to 8 deterministic n8n workflows plus one composite Supabase query plus a Markdown template node. No reasoning load exists at 10-30 daily touches. Hermes agents are deferred until volume or scoring complexity demands them. The agent plan adds model API cost, latency, and new failure surfaces with no benefit at current volume.

---

## Build spec

**Workflow inventory**

| Workflow | Trigger | Auto-send? |
|---|---|---|
| PROD-CRM-LeadIntake-ScoreRoute-v1.0 | Webhook: form POST | Booking-link email: yes. SMS: no. Routing writes: yes. |
| PROD-OPS-ShiftPrep-CommandBrief-v1.0 | Schedule 09:30 ET, M-F | Yes, to operator only. |
| PROD-OPS-CallWindow-LocalTimeCheck-v1.0 | Sub-workflow call | Blocks send; no outbound. |
| PROD-CRM-DocChase-StageWatch-v1.0 | Supabase row-change webhook on stage=S3 | No. Writes draft to pending_actions. |
| PROD-CRM-DocChase-Scan-v1.0 | Schedule every 2h, 10:00-19:00 ET | No. Queues to pending_actions. |
| PROD-CRM-PendingActions-Fire-v1.0 | Schedule every 5 min | Executes operator-approved rows only. |
| PROD-LEADS-UCCPull-EnrichDedupeQueue-v1.0 | Manual (schedule after week 3) | No. Writes to ucc_queue for review. |
| PROD-OPS-InboundReply-ParseOptOut-v1.0 | Webhook: inbound email | Suppression write: yes. No outbound. |
| PROD-OPS-EndOfDay-Report-v1.0 | Schedule 19:15 ET, M-F | Yes, to operator only. |
| PROD-CRM-Notes-Cleanup-v1.0 | Manual: operator pastes notes | No. Draft to pending_actions for review. |

**Approvals: what auto-sends vs requires operator action**

Auto-execute without approval: Supabase writes from scoring, opt-out suppression writes, call-window deferred-action writes, internal notifications, command briefs, reports.

Requires operator action in `pending_actions` before any outbound fires: document-chase emails, any merchant-facing email generated by n8n, any SMS (post-consent only), CRM note saves from note cleanup. The operator flips `status=approved` in Supabase Table Editor. `PROD-CRM-PendingActions-Fire-v1.0` polls every 5 minutes and executes.

**Supabase tables required before any workflow goes live:** `leads`, `pending_actions`, `workflow_errors`, `intake_errors`, `deferred_actions`, `ucc_queue`, `config` (with `system_halt` boolean). Flag to the data architect lane.

**Host recommendation: small VPS, not a mini PC.**

A home mini PC is dark and unrecoverable during ET work hours if the Israeli power or ISP fails. The worst failure window is exactly 10:00-19:00 ET. A Hetzner CX22 or Contabo S (6-10 USD/month) runs in a data center with a 99.9 percent uptime SLA and is SSH-accessible from any device. n8n runs self-hosted on the VPS via Docker, eliminating the n8n Cloud execution cap concern entirely. Total infrastructure cost is under 20 USD/month.

**Resilience mechanisms**

Every workflow has an explicit error branch that writes to `workflow_errors` (workflow name, version, entity ID, error class, short note, timestamp) and sends an operator email alert. No silent failures.

Retries: webhook-triggered workflows return HTTP 200 immediately and process asynchronously, preventing timeout-induced duplicate retries from the sender. HTTP nodes (Apollo, Cal.com, SMTP) use 3 retries with 15-second backoff. Records that fail enrichment or delivery land in `pending_actions` with `type=manual_review`.

Idempotency: LeadIntake checks `form_submission_id` before insert. DocChase-StageWatch checks `s3_entered_at IS NULL`. PendingActions-Fire checks `status=approved` and sets `status=processing` atomically before executing.

Kill switch: `config.system_halt = true` halts all outbound within one 5-minute polling cycle. Operator sets it from the Supabase dashboard. No VPS access required.

Uptime monitoring: UptimeRobot free tier pings the n8n health endpoint every 5 minutes. On failure, it sends a Telegram alert. The operator can SSH-restart the process from a phone. Without this monitor, a process crash during work hours is invisible until the command brief fails to arrive.

---

## Over-built or cut

**Three-agent architecture:** CUT. Replaced by 10 deterministic workflows and one Supabase ranked-queue query.

**Always-running Hermes agent processes:** DEFERRED. No reasoning task exists at current volume.

**Model calls in scoring, routing, or queue ranking:** CUT. SQL and a Code node are sufficient and have zero model API cost.

**Separate Sales Intelligence agent:** CUT. Brief is a templated SQL result. After-call cleanup is one manual-trigger DeepSeek call.

**Automated Tier 3 nurture sequences:** CUT per strategy. A single 90-day re-ping entry is written by the stage-watch workflow; no drip sequence.

**A/B testing infrastructure:** CUT per strategy.

---

## Top risks

**Risk 1: VPS or n8n process crashes during 10:00-19:00 ET with no recovery signal.**
A webhook drop in this window means missed lead intakes with no notification. The operator is selling and cannot monitor server health. Mitigation: UptimeRobot 5-minute ping with Telegram alert plus Docker restart policy `always`. Without this, the operator learns of the outage only when the shift-end report does not arrive, hours after the damage.

**Risk 2: Supabase row-change webhook fires duplicate events on S3 transition, causing double document-chase drafts queued to `pending_actions`.**
If the operator approves both, the merchant receives two document-request emails within minutes. This is unprofessional and potentially damaging to a warm deal. Mitigation: the `s3_entered_at IS NULL` idempotency guard in the stage-watch workflow is the only protection. It must be tested explicitly with a duplicate-event test before production.

**Risk 3: Booking embed fails silently, no fallback exists, and a high-score inbound lead is lost with no record in Supabase.**
The plain-form fallback and the command-brief gap-detection query (route=booking_link and booking_created=false older than 4 hours) are the two mitigations. If the front-end lane does not build the fallback before the page goes live, this risk is unmitigated and the gating-page conversion rate is invisible.

---

## Constraint flags

**TCPA consent gates all SMS.** `opted_out` and `consent_sms` checks are the first two conditions in every outbound workflow. The opt-out suppression auto-write has no human-approval step because latency creates compliance exposure. Attorney review of consent copy must complete before the page goes live and any outreach fires. This is a legal precondition, not a best practice.

**State-cleared filter gates UCC outreach.** A `state_cleared` boolean per lead must be set (manually or by a state-filter node in the UCC workflow) before any touch is created for that merchant. State broker registration and commercial financing disclosure requirements (CA SB 1235, NY CFDL, TX HB 700) are unverified per state. Outreach to uncleared states must not fire. Flag to the compliance lane.

**DeepSeek access to PII.** All 10 n8n workflows are model-free; they do not send PII to any model. The one model call (note cleanup via PROD-CRM-Notes-Cleanup-v1.0) receives operator-pasted call notes, not raw bank statements or owner SSN/EIN. If bank statement data is ever passed to a model, the default must change from DeepSeek to a US-hosted model. Flag to the data architect lane.

**Budget.** VPS at 6-10 USD/month plus SMTP (Postmark free tier) plus UptimeRobot (free) totals under 15 USD/month for automation infrastructure. Well within the 300 USD cap.

---

## Cross-lane flags

**Front-end/UX lane:** plain-form fallback must POST to the exact n8n webhook URL with all required fields. The page team must confirm field mapping before `PROD-CRM-LeadIntake-ScoreRoute-v1.0` is finalized. Without this, the fallback path will produce validation errors in n8n.

**Data architect lane:** `pending_actions`, `workflow_errors`, `intake_errors`, `deferred_actions`, `ucc_queue`, and `config` tables must be defined with correct column types and row-level security before any workflow goes to production. The n8n service account needs write access to `leads`, `pending_actions`, `workflow_errors`, `intake_errors`, `deferred_actions`, `ucc_queue`, and `config.system_halt` only.

**Compliance lane:** TCPA one-to-one consent (Jan 2025 FCC rule, Eleventh Circuit challenge) applicability to UCC-sourced sole-proprietor cell contacts is unresolved. Until cleared, the UCC workflow writes records for operator review only; no outbound is queued automatically.

**Integration lane:** Cal.com free-tier outgoing webhook support must be confirmed before `PROD-OPS-ShiftPrep-CommandBrief-v1.0` is finalized. If Cal.com free tier does not fire booking-creation webhooks, the booking-created flag cannot be set automatically and the queue ranking loses its strongest warmth signal.

---

## Facts to verify

1. Cal.com free tier: does it support outgoing webhooks on booking creation? If not, polling the Cal.com API on a schedule is the fallback, adding 2-5 minute latency to queue promotion.
2. Supabase database webhooks (via pg_net or Realtime): confirm whether a row-change webhook on `leads.stage` reliably fires an n8n HTTP call, or whether a Supabase Edge Function is required as an intermediary. The reliability of this path determines the doc-chase trigger architecture.
3. Postmark or equivalent inbound parse: confirm that inbound email can be forwarded to an n8n webhook and that forwarding latency is under 5 minutes, which is the acceptable window for opt-out suppression to activate before any subsequent automated send.
4. Apollo free-tier rate limit: confirm the per-day enrichment cap against an expected 50-100 UCC records per weekly batch. If the batch exceeds the daily cap, enrichment must be spread across multiple days via a looping workflow.
5. Hetzner CX22 availability in a US-East region: confirm co-location proximity to the Supabase US-East project to minimize round-trip latency on Supabase queries in scheduled workflows.
6. n8n self-hosted memory footprint on a 2 vCPU / 4 GB RAM VPS at 10 concurrent workflow executions: verify this is sufficient before committing to the CX22 tier.
