# Pass 3 Audit: Autonomous Optimization Architect

**Lane:** Model routing policy, token tracking, the 300 USD/month hard cap as an enforced guardrail, and the DeepSeek-on-PII residency risk.

---

## Implementing the strategy

**Tactic 3: Warmth scoring is deterministic, no model call.**
Build confirmed. No model needed. A Supabase function or n8n expression evaluates the hard-coded point values and writes the tier field. Zero token cost. Any proposal to add a model call here is over-built and rejected.

**Tactic 4: Discovery note cleanup and document-request drafting.**
Build: after-call note cleanup routes to Claude Sonnet 3.5 (now claude-sonnet-4-5 or equivalent at roughly 3 USD per million output tokens). At 10-30 calls per day and roughly 300 output tokens per cleanup, daily spend is under 0.05 USD. Document-request draft generation goes to the same model in the same n8n node, same call, same cost envelope. This is the correct lane for Claude: judgment, compliance-sensitive language, and structured output that a funder or attorney might read.

**Tactic 4 also: Lead briefs.**
Build: Tier 1 lead briefs (why-call, risk flags, opener) route to Claude Sonnet 3.5. At 30 Tier 1 briefs per day and roughly 400 output tokens each, daily spend is under 0.04 USD. Both brief and note cleanup combined stay under 0.09 USD per day, roughly 2.70 USD per month.

**Plan section 8: DeepSeek as default model.**
Partial build, with a hard carve-out. DeepSeek (V3 or R2-lite via API) costs roughly 0.14 USD per million input tokens and 0.28 USD per million output tokens as of mid-2025. It is appropriate for: formatting raw web data into structured JSON for enrichment, non-sensitive template assembly (email subject line variants, no merchant name or financials), and internal logging summaries with no PII. It is NOT acceptable for: bank statement parsing, merchant revenue or deposit figures, owner name or phone, UCC data that includes business financials, or anything that would constitute "personal data" under a US-facing data processing agreement. DeepSeek is a Chinese-hosted model with no published US data residency, no SOC 2, and no contractual data processing agreement that covers US business PII or financial data. Routing those payloads through DeepSeek is a regulatory and reputational risk that is not worth saving 2 USD per month. Decision: DeepSeek handles only non-PII formatting tasks. All PII and financial data routes to Claude or OpenAI, or to a local model on the mini PC.

**Local model on mini PC for cheap formatting and parsing.**
Build: run Ollama with Mistral 7B or Phi-3-mini. Use cases: stripping HTML from scraped pages before any enrichment call, normalizing address formats, deduplication checks, parsing UCC XML fields into structured rows. Cost: zero marginal token cost after hardware. Acceptable latency for background jobs (not synchronous with a user action). Data never leaves the local machine. This is the correct answer for bank-statement field extraction where DeepSeek is excluded and Claude cost-per-call is unnecessary for a simple structured parse.

**Plan section 10: Budget.**
Build a per-task token budget table enforced in n8n before each model call. Each task type has a max-input-token limit and a max-output-token limit. If the constructed prompt exceeds the input limit, the node fails closed and logs the overflow rather than sending an oversized (and over-priced) request.

---

## Build spec

### Routing policy (five tiers, fixed)

| Task | Model | Rationale |
|---|---|---|
| Warmth scoring | No model, Supabase function | Deterministic, zero cost |
| HTML cleanup, address normalization, deduplication | Local Ollama (Mistral 7B) | No PII, zero marginal cost |
| Non-PII template formatting (subject lines, internal summaries) | DeepSeek V3 API | Cheap, no PII in payload |
| Bank statement field extraction, revenue/deposit parsing | Local Ollama or Claude | PII + financial data, cannot route to DeepSeek |
| Lead briefs, objection handling, note cleanup, document-request drafts | Claude Sonnet 3.5 (or GPT-4o-mini as fallback) | Judgment, compliance language |
| Escalation or complex objection scripts | Claude Opus (on-demand only, not scheduled) | High judgment, rare, budget-gated |

### Token budget enforcement (n8n)

Each workflow node that calls a model receives a pre-call check:

1. Measure prompt token count using a tiktoken-compatible library or a character-count heuristic (4 chars per token as a safe upper bound).
2. Compare against the task ceiling (brief: 2,000 input / 500 output; note cleanup: 1,500 input / 400 output; formatting: 500 input / 200 output).
3. If over ceiling: log the overflow to a Supabase table, skip the model call, surface a manual-review flag, and continue the workflow without failing the lead.
4. Never retry a failed model call more than once. No unbounded retry loops. One attempt, one fallback, then manual queue.

### Billing caps and alerts

- Set a hard spend cap on the Anthropic console: 10 USD/month. Alert at 7 USD.
- Set a hard spend cap on the OpenAI console (if used as fallback): 5 USD/month. Alert at 3 USD.
- DeepSeek: set a 2 USD/month prepaid credit ceiling. No auto-reload.
- n8n: log each model call to a `model_usage` table in Supabase with task type, model, input tokens, output tokens, and estimated cost. A daily n8n scheduled job sums the month-to-date cost and writes it to a `budget_status` row.
- If month-to-date model spend exceeds 18 USD (the model spend budget line), n8n sets a `model_blocked` flag to true. All model-call nodes check this flag first. If true, they skip the model call and enqueue a manual-review task. This is fail-closed: the system degrades gracefully to human review rather than spending past the cap.
- Circuit breaker: if any single model call returns HTTP 429 or 402 three times in one hour, disable that provider for the rest of the day and switch to the next tier down (Claude to OpenAI, OpenAI to local).

### Monthly cost breakdown within 300 USD

| Line item | Est. monthly cost |
|---|---|
| Claude API (briefs, note cleanup, doc drafts) | 3-8 USD |
| DeepSeek API (non-PII formatting only) | 1-2 USD |
| OpenAI API (fallback, rare) | 0-3 USD |
| Supabase (free tier, US region) | 0 USD |
| n8n (self-hosted on mini PC or free cloud tier) | 0-20 USD |
| UCC data vendor (sub-100 USD cap, settled) | 50-80 USD |
| Cal.com (free tier) | 0 USD |
| Qualification page host (Vercel free or Netlify free) | 0 USD |
| Mini PC electricity (always-on, 10W idle) | 1-2 USD |
| Apollo free tier enrichment | 0 USD |
| Buffer | 180-240 USD |

Total estimated spend: 55-115 USD per month. Hard cap is 300 USD. Buffer is intentional: this is a 90-day run with unknown UCC vendor final pricing.

---

## Over-built or cut

- Shadow testing across providers: cut. Volume is 10-30 model calls per day. There is no statistically meaningful signal at this volume to auto-promote a new model. Manual quarterly review is sufficient. The shadow-testing framework from this persona's toolkit is designed for thousands of calls per day.
- LLM-as-a-Judge grading loops: cut for the same reason. One person can read 30 note-cleanup outputs per day. Automated grading adds infrastructure cost and complexity with no return at this scale.
- Autonomous traffic re-weighting: cut. Hard-coded routing table, updated manually when the operator decides to change it. Adds zero maintenance burden.
- Gemini Flash or other providers: not worth adding a third model API account for marginal savings at this volume. Two providers (Claude primary, OpenAI fallback) plus local is the right ceiling.

---

## Top risks

1. **Mini PC offline during US work hours kills all model calls and n8n workflows.** If the mini PC is the always-on host for n8n and Ollama and it loses power or internet, every event-triggered workflow silently fails during the prime call window (10:00-14:00 ET). Mitigation: host n8n on the free cloud tier (n8n.cloud) for event handling and scheduled jobs; keep Ollama local for offline-tolerant background tasks only. The operator's shift is 10:00-19:00 ET. An outage in that window costs real deals.
2. **DeepSeek payload contamination.** A developer or future change accidentally passes a merchant's bank statement, owner name, or revenue figures to a DeepSeek call. No technical guard exists against this today beyond routing policy. Mitigation: add a payload-scrubbing middleware node in n8n before every DeepSeek call that checks for a PII flag field in the Supabase record and hard-blocks the call if the flag is set. Treat this as a required gate, not a nice-to-have.
3. **Token cost spike from a malformed prompt in a looping workflow.** A bug in the note-cleanup template sends an oversized context (for example, the full email thread instead of the operator's notes) to Claude on every no-answer event. At low volume this is manageable, but a misconfigured trigger that fires 50 times per hour would exhaust the monthly model budget in one shift. Mitigation: the per-task input token ceiling (step 1 of the enforcement spec above) catches this before the call is made. The `model_blocked` flag catches any bypass. Both must be in place at launch.

---

## Constraint flags

- **DeepSeek PII:** Do not route any field containing merchant legal name, owner name, phone, email, EIN, revenue, deposits, bank name, or UCC debtor data through DeepSeek until the operator has a signed data processing agreement that specifies US data residency and a documented retention and deletion policy. This is a legal gate, not a preference. Until that DPA exists, DeepSeek handles only fully anonymized or non-PII content. A safe test: could this payload appear in a loan application? If yes, it does not go to DeepSeek.
- **300 USD hard cap:** The model spend ceiling is 18 USD/month (roughly 6 percent of total budget, matching the low-volume profile). The `model_blocked` flag is the enforcement mechanism. This must be configured before any live model call, not after.
- **Supabase US region:** Bank statement data and merchant PII must reside in a US region (us-east-1 or us-west-1). Verify the project region in the Supabase console before storing any financial data. Column-level encryption for bank statement fields is a good practice but not a day-one blocker; row-level security with strict policies is the minimum.

---

## Cross-lane flags

- The qualification page host (Vercel vs Netlify vs mini PC) affects whether n8n webhooks can reach it reliably during US hours. Infrastructure lane should decide this and confirm the webhook endpoint is publicly reachable with a static URL before the page goes live.
- TCPA consent capture on the page, and whether the consent language covers SMS and phone calls to sole proprietors, is a compliance lane question. Model routing does not touch it, but the consent flag stored in Supabase must be readable by n8n before any automated outreach is triggered.
- The queue formula weights (Warmth 0.40, EV Tier 0.35, Clawback Survivability 0.15, Recency 0.10) are computed in Supabase, not by a model. The pipeline analyst lane owns the formula. This lane enforces that no model call substitutes for it.
- Cal.com vs Calendly choice affects whether the booking webhook fires reliably into n8n. Either works at free tier; the infrastructure lane should pick one and lock it before the page is built.

---

## Facts to verify

- Anthropic console: confirm hard spend caps are available and enforceable (not just email alerts) at the 10 USD/month level. As of early 2025 Anthropic offered spend limits; verify this is still the case and that the cap blocks API calls, not just sends an email.
- DeepSeek API terms of service: does the current agreement include a data processing addendum, EU-SCCs, or any US data residency clause? If not, PII exclusion is mandatory and the routing policy above stands.
- n8n cloud free tier execution limits: the free tier caps at 5 active workflows and 2,500 executions per month. At 10-30 leads per day with multiple workflow steps per lead, this may be exhausted in week two. Either self-host n8n on the mini PC (free but availability-dependent) or pay 20 USD/month for the Starter tier. Budget has room for this.
- Ollama + Mistral 7B on the mini PC: verify the machine has at least 8 GB RAM and that inference runs in under 10 seconds for a 500-token parsing job. If the mini PC is underpowered, Phi-3-mini (3.8B) runs on 4 GB and is fast enough for formatting tasks.
- OpenAI GPT-4o-mini pricing: as of mid-2025 it is roughly 0.15 USD per million input tokens and 0.60 USD per million output tokens, making it a viable fallback below Claude for note cleanup if Anthropic spend limits are approached.
