# Pass 3 Audit: Email Intelligence Engineer

**Lane:** Parsing inbound email and SMS replies into structured, reasoning-ready records that drive pipeline state transitions.

---

## Implementing the strategy

**Tactic 2 (consent capture on the page):** The page fires a webhook to n8n on submission. The reply parser must read the consent timestamp and channel scope out of the Supabase `leads` row before sending any reply. No email or SMS reply is sent unless the consent record exists and has not been revoked. This is not optional logic; it gates every outbound trigger downstream.

**Tactic 3 (deterministic routing):** Inbound reply classification is the mirror of the outbound scoring logic. The parser produces one of six intent labels and routes deterministically. No model makes a routing decision; the model only extracts the label and entities. The n8n workflow reads the label and writes the next-action field.

**Tactic 6 (document collection as stage trigger):** A reply that contains a forwarded bank statement or an attachment with mime type PDF or image advances the lead from S3 to S4 in Supabase. This is the highest-value routing rule in the lane. The trigger must fire within two minutes of email receipt so the SDR sees the stage change before the next action fires.

---

## Build spec

### Ingestion

- Gmail API webhook (Pub/Sub push) or IMAP IDLE polling every 60 seconds on the SDR's inbox. n8n handles the trigger. Cost: free tier.
- SMS replies arrive via Twilio webhook directly into n8n. Same classifier, shorter text.
- Every raw message is written to `raw_replies` in Supabase (message_id, from, subject, body_text, attachment_names, received_at, lead_id) before any parsing. This is the audit trail and the failure recovery point.

### Parser and classifier

One n8n function node calls a model with a structured prompt. Input: stripped plain-text body (quoted reply content removed, signature removed), sender email or phone, attachment list (names only, not content), subject line. Output: JSON.

```json
{
  "intent": "interested | not_now | opt_out | wrong_number | document_attached | objection_question",
  "entities": {
    "business_name": "string | null",
    "requested_amount": "number | null",
    "urgency": "asap | this_week | this_month | null",
    "objection_text": "string | null",
    "question_text": "string | null"
  },
  "has_attachments": true,
  "attachment_count": 1,
  "confidence": 0.0
}
```

Model choice: DeepSeek for plain-text replies. DeepSeek is cheap and fast and the text is low-sensitivity at this stage: the body is stripped plain text with no financial figures. Do NOT pass attachment content to any model. Attachment names (e.g., "bank_statement_march.pdf") are passed only as strings for label inference.

**Bank statement content is PII and financial data. It must not be sent to DeepSeek or any non-US model with unclear retention.** If confidence on the intent label is below 0.70 or the reply contains an unrecognized structure, route to the SDR's "needs human review" queue in Supabase and send no automated action.

### Routing rules (deterministic after classification)

| Intent | Action |
|---|---|
| `interested` | Set lead status to "warm_reply", write next-action "call today", fire SDR alert |
| `not_now` | Write follow-up date (+30 days), set status "nurture", no immediate contact |
| `opt_out` | Set `opted_out = true`, `opted_out_at = now()`, `opted_out_channel = email/sms`, suppress all future sends immediately, log to `compliance_events` table |
| `wrong_number` | Flag lead as bad contact data, pause all outreach, write note "owner says wrong number or not owner" |
| `document_attached` | If lead is in S3, advance to S4, write `s3_exit_at`, fire "documents received" alert to SDR, queue attachment download |
| `objection_question` | Write objection/question text to lead notes, set next-action "respond to objection", queue for SDR review |

### Opt-out handling (compliance-critical)

Opt-out triggers are: "STOP", "unsubscribe", "remove me", "do not contact", "wrong number" (treated as implicit opt-out from this number), or any variant. The classifier must catch these before any other logic runs. The n8n node checks for opt-out keywords in raw text before even calling the model, as a hardcoded guard. On opt-out:

1. `opted_out = true` written to `leads` row within one n8n execution (under 10 seconds from receipt).
2. A row is inserted into `compliance_events` with the raw message body, timestamp, channel, and the automated action taken.
3. All pending n8n executions for that lead_id that have not yet sent are cancelled via a lead status check at send time. Any execution that reads `opted_out = true` aborts and logs.
4. No confirmation message is sent to the opt-out unless the SDR explicitly approves one; auto-sending a confirmation risks an unwanted contact on a suppressed number.

### Attachment handling

Attachments are downloaded by n8n to a temporary local path (on the always-on VPS or mini PC). They are then uploaded to Supabase Storage in a private bucket (`bank-statements`) with row-level security scoped to the authenticated SDR user only. The file path and upload timestamp are written to `lead_documents`. The raw content is never passed to any external model. If the brokerage has a lender portal for document upload, the SDR receives an alert to manually upload; no automated third-party transfer.

Attachment type inference uses the filename extension and MIME type header only. If the filename contains "statement", "bank", "account", or common bank name strings, label it `bank_statement` in the `lead_documents` row.

### Where this runs

n8n on the always-on host (VPS recommended over home mini PC for uptime reliability during Israel working hours). n8n handles the Gmail/Twilio webhook, the classifier call, the routing logic, and the Supabase writes. No separate service needed. The model API call (DeepSeek) is the only external dependency per reply.

### Failure modes and fail-safe behavior

- Gmail Pub/Sub drops a message: IMAP IDLE polling as fallback every 60 seconds catches it.
- DeepSeek API is down: the n8n function catches the error, writes the raw reply to `raw_replies` with `parse_status = "failed"`, and fires an SDR alert. No routing action is taken. The SDR reviews manually. No reply is suppressed or lost.
- Opt-out keyword check always runs before the model call, so a model outage never delays an opt-out.
- Attachment download fails: the stage transition to S4 still fires (the email itself is evidence of intent), but the attachment is flagged as "download_failed" for the SDR to follow up.

---

## Over-built or cut

**Cut: semantic search or embedding over replies.** At 10-30 replies per day, a six-label classifier is sufficient. No vector index needed.

**Cut: thread reconstruction.** Inbound replies are single-message responses to outbound. Full thread topology reconstruction is enterprise-scale overhead not needed here. Strip quoted content and process the new body only.

**Cut: sentiment scoring beyond intent label.** The six intents are sufficient for routing. Nuance lives in the objection text field, which the SDR reads directly.

**Cut: automated response generation.** The plan calls for SDR approval before any substantive reply. Do not auto-send anything except opt-out confirmation if required by law (verify this). The parser feeds the SDR dashboard; it does not close the loop autonomously.

**Cut: local model for classification.** DeepSeek is cheap enough (sub-cent per reply at this volume) and more accurate than a self-hosted small model on short ambiguous business text. A local model for formatting (e.g., after-call note cleanup) is fine, but not for intent classification where a wrong label on an opt-out is a compliance event.

---

## Top risks

1. **Opt-out missed during a model or n8n outage, causing a suppressed lead to receive another touch.** Mitigation: the hardcoded keyword check runs before the model call and writes directly to Supabase; the model outage path still fires the keyword check. Test this failure path explicitly before going live.

2. **Bank statement content leaking to a non-US model via an attached-file parsing step added later.** The current spec never passes attachment content to any model. The risk is that a future "convenience" change adds OCR or content extraction and passes the result to DeepSeek. The `bank_statement` label on the `lead_documents` row must be used as a hard gate in any future extraction step: if `document_type = "bank_statement"`, extraction must use a US-hosted model (Claude via Anthropic US endpoints) or a local model only.

3. **n8n execution backlog during high-volume bursts causes a reply to sit unprocessed for minutes during the prime call window, making the SDR miss a hot inbound.** Mitigation: set n8n workflow concurrency to process reply workflows immediately on trigger, not queued. At 10-30 touches per day, inbound volume is low enough that this is not a persistent problem, but a missed bank-statement reply during the call window costs a deal.

---

## Constraint flags

- **TCPA and SMS opt-out (compliance gate):** SMS STOP must be honored immediately per TCPA and carrier rules. The hardcoded keyword check for SMS replies is non-negotiable before go-live. Verify with counsel whether a written confirmation of opt-out is required on SMS (carriers typically handle STOP auto-reply, but confirm Twilio's behavior for the account type).
- **Bank statement PII and model routing (compliance gate):** DeepSeek's data retention policy for API calls is not publicly confirmed as US-only or zero-retention. Do not pass bank statement content to DeepSeek under any path. This gates any future OCR or extraction feature on bank documents to Claude (Anthropic US) or a local model.
- **Supabase US region (verify):** Confirm the Supabase project is provisioned in us-east-1 or us-west-2. Bank statement files in Supabase Storage must not sit in a non-US region given the PII sensitivity.
- **Budget:** DeepSeek API cost for reply classification is negligible at this volume (under 2 USD/month). Twilio SMS inbound webhook is free on the inbound side. Gmail Pub/Sub is free. No budget risk in this lane.

---

## Cross-lane flags

- **After-call note cleanup** (plan section 6): The SDR pastes rough notes and receives a structured CRM record plus a document-request draft. This is a model call on potentially sensitive call notes. The Sales Intelligence Engineer lane owns this. Flag: if notes include financial figures or merchant names, the same model-routing rule applies (no DeepSeek for PII-heavy content).
- **Queue ranking and warmth score update on reply receipt:** When the parser writes `status = "warm_reply"` or `s3_exit_at`, the queue rank for that lead must recompute. The Pipeline Analyst lane owns the ranking formula. This lane only writes the trigger fields; it does not rerank.
- **Outbound send suppression on opt-out:** The n8n outbound cadence workflows must check `opted_out` before every send. This lane writes the flag; the Sales Outreach lane must read it. Verify the outbound workflows perform this check.
- **Booking webhook as an inbound reply type:** A Cal.com or Calendly booking confirmation arrives as an email to the SDR inbox. The parser must recognize the booking confirmation sender domain and route it as `interested` with an entity of `booking_confirmed = true`, not classify it as a generic reply. This crosses into the Warm Lead Engine lane.

---

## Facts to verify

1. Twilio's automatic STOP handling for SMS: does Twilio's platform auto-suppress and auto-reply on STOP before the webhook fires, or does the webhook fire first and the application must suppress? This changes whether the n8n keyword check is a redundant safety net or the primary mechanism.
2. DeepSeek API data retention and US data residency: public documentation is sparse. If retention is not confirmed as zero or US-only, route all reply classification through Claude (cost impact: roughly 0.50 to 2 USD/month at this volume, still within budget).
3. Gmail Pub/Sub push subscription reliability and whether the always-on VPS needs a public HTTPS endpoint to receive pushes (yes it does), and whether the IMAP IDLE fallback is needed given the VPS has a stable IP.
4. Supabase Storage private bucket and RLS: confirm that the `bank-statements` bucket is not publicly accessible by default and that the RLS policy requires authenticated access scoped to the SDR user.
5. Whether a forwarded bank statement reply (the merchant forwards from their bank's email address) retains the original bank email as a nested MIME part or arrives as a plain attachment. This changes the attachment detection logic: a nested forward may not have a file extension and the MIME type may be `message/rfc822`, not `application/pdf`.
