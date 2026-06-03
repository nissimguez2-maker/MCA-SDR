# Pass 3 Backend Architect Audit: Supabase Schema and Event Model

**Lane:** Supabase schema, tables, events, and source-of-truth design for the MCA SDR pipeline.

---

## Implementing the strategy

**Tactic 1 (signal outreach, single CTA to page):** No backend work needed beyond a leads table that holds source_type and the UCC/prior-funded signal fields. n8n posts the submission; Supabase stores it. No agent required.

**Tactic 2 (one qualification page, consent capture):** Page submission writes one row to `leads`. Fields: `consent_timestamp TIMESTAMPTZ NOT NULL`, `consent_scope TEXT NOT NULL` (exact checkbox text, versioned), `tcpa_ip INET`, `tcpa_user_agent TEXT`. These are the consent defensibility fields. Storing them here, in-row, is the right call. Do not normalize consent into a separate table; join cost is not worth it at this volume.

**Tactic 3 (deterministic warmth scoring in n8n):** n8n computes the score from hard-coded rules and writes `warmth_score SMALLINT` and `warmth_computed_at TIMESTAMPTZ` to `leads`. Supabase holds the result, not the logic. Correct separation.

**Tactic 4 (discovery call, tier assignment):** After the call, n8n or the operator writes `tier SMALLINT CHECK (tier IN (1,2,3,4))` (4 = Pause) and `call_notes_raw TEXT` to `leads`. Stage moves from S2 to S3 on tier assignment if docs are requested.

**Tactic 5 (queue formula):** The daily queue is a Supabase view or a single n8n SQL query. No separate table needed. Confirm the queue computation runs at shift start, not continuously.

**Tactic 6 (document collection as stage-transition trigger):** S2-exit writes `stage = 'S3'` and `s3_entered_at TIMESTAMPTZ`. A Supabase database trigger (or n8n webhook on row update) fires the document-request draft and sets `doc_chase_due TIMESTAMPTZ = NOW() + INTERVAL '48 hours'`. A scheduled n8n job (every 30 min during US hours) queries `WHERE stage = 'S3' AND doc_chase_due < NOW() AND docs_received = FALSE` and sends the chase. Simple, deterministic, always-on.

---

## Build spec

### Tables

```sql
-- Core: one row per lead from first touch onward
CREATE TABLE leads (
  id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_type               TEXT NOT NULL,          , 'prior_funded','ucc','inbound','company'
  business_name             TEXT,
  owner_name                TEXT,
  phone                     TEXT,                   , encrypted, see security section
  email                     TEXT,                   , encrypted
  monthly_revenue_usd       INTEGER,                , encrypted
  time_in_business_months   SMALLINT,
  stack_count               SMALLINT DEFAULT 0,
  funded_date               DATE,
  offer_term_months         SMALLINT,
  last_contact_date         DATE,
  warmth_score              SMALLINT,
  warmth_computed_at        TIMESTAMPTZ,
  distress_flag             BOOLEAN DEFAULT FALSE,
  tier                      SMALLINT CHECK (tier IN (1,2,3,4)),
  stage                     TEXT NOT NULL DEFAULT 'S1'
                              CHECK (stage IN ('S1','S2','S3','S4','S5','S6','S7')),
  call_notes_raw            TEXT,
  doc_chase_due             TIMESTAMPTZ,
  docs_received             BOOLEAN DEFAULT FALSE,
  consent_timestamp         TIMESTAMPTZ,
  consent_scope             TEXT,
  tcpa_ip                   INET,
  tcpa_user_agent           TEXT,
  clawback_window_end       DATE,
  created_at                TIMESTAMPTZ DEFAULT NOW(),
  updated_at                TIMESTAMPTZ DEFAULT NOW()
);

-- Stage definitions (reference only, no join needed in hot paths)
-- S1: in cadence / no page visit
-- S2: page completed, consent captured, score computed
-- S3: discovery call done, doc request sent, 48h chase armed
-- S4: docs received and complete
-- S5: submitted to funder
-- S6: approved / offer out
-- S7: funded or closed/dead

-- Event log: immutable append for audit and coaching
CREATE TABLE activities (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id     UUID REFERENCES leads(id) ON DELETE CASCADE,
  event_type  TEXT NOT NULL, , 'call','voicemail','email','stage_change','score_update','doc_request','doc_chase','consent_captured'
  event_at    TIMESTAMPTZ DEFAULT NOW(),
  payload     JSONB,         , stage_from/stage_to, score_before/after, touch number
  created_by  TEXT           , 'n8n','operator','trigger'
);

-- Funder submissions: one row per submission per funder
CREATE TABLE funder_submissions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id         UUID REFERENCES leads(id) ON DELETE CASCADE,
  funder          TEXT NOT NULL,
  submission_date DATE NOT NULL,
  stips           TEXT[],            , outstanding stips list
  stips_cleared   BOOLEAN DEFAULT FALSE,
  offer_expiry    DATE,
  funded_amount   INTEGER,           , encrypted
  factor_rate     NUMERIC(5,4),
  funded_date     DATE,
  clawback_days   SMALLINT,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### Indexes

```sql
CREATE INDEX idx_leads_stage_tier ON leads(stage, tier) WHERE stage NOT IN ('S7');
CREATE INDEX idx_leads_doc_chase ON leads(doc_chase_due) WHERE stage = 'S3' AND docs_received = FALSE;
CREATE INDEX idx_activities_lead ON activities(lead_id, event_at DESC);
CREATE INDEX idx_funder_submissions_lead ON funder_submissions(lead_id);
```

### S2-exit trigger (Supabase database function)

```sql
CREATE OR REPLACE FUNCTION on_stage_change()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
  IF NEW.stage != OLD.stage THEN
    INSERT INTO activities(lead_id, event_type, payload, created_by)
    VALUES (NEW.id, 'stage_change',
            jsonb_build_object('from', OLD.stage, 'to', NEW.stage),
            'trigger');
    IF NEW.stage = 'S3' THEN
      NEW.doc_chase_due = NOW() + INTERVAL '48 hours';
    END IF;
  END IF;
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$;

CREATE TRIGGER trg_stage_change
BEFORE UPDATE ON leads
FOR EACH ROW EXECUTE FUNCTION on_stage_change();
```

n8n polls `activities WHERE event_type = 'stage_change' AND payload->>'to' = 'S3' AND created_at > NOW() - INTERVAL '1 minute'` on a 1-minute schedule to pick up new S3 entries and fire the document-request draft. The 48-hour chase is handled by a separate n8n schedule querying `idx_leads_doc_chase`.

### Queue view (shift-start query)

```sql
CREATE VIEW daily_queue AS
SELECT
  id, business_name, owner_name, phone, tier, stage,
  warmth_score,
  clawback_window_end,
  (warmth_score * 0.40
   + CASE tier WHEN 1 THEN 35 WHEN 2 THEN 20 ELSE 5 END * 0.35
   + CASE WHEN clawback_window_end > CURRENT_DATE THEN -4 ELSE 10 END * 0.15
   + CASE WHEN last_contact_date >= CURRENT_DATE - 3 THEN 10 ELSE 0 END * 0.10
  ) AS queue_score
FROM leads
WHERE stage IN ('S1','S2','S3') AND tier IN (1,2,3)
ORDER BY queue_score DESC;
```

### Security model

Column-level encryption is required for: `phone`, `email`, `monthly_revenue_usd`, `funded_amount` (in funder_submissions). Use Supabase's `pgsodium` extension (available on all Supabase projects) with `security.encrypt_text` on these columns. Alternatively, encrypt in n8n before insert using AES-256 and store ciphertext; decrypt only in the n8n layer, never in a DeepSeek-accessible context.

Row-level security is required on all three tables. The only Supabase service-role key goes to n8n, running on a self-controlled host. DeepSeek must never receive a service-role key or raw PII columns. If DeepSeek is used for note cleanup, pass only the `id` and anonymized text; never pass `phone`, `email`, or revenue fields.

Supabase project must be created in `us-east-1` or `us-west-1`. Verify region at project creation; it cannot be changed after. Bank statements are never stored in Supabase. They go to a US-region S3 bucket or Supabase Storage with a short-lived signed URL, not as raw blobs in the database.

---

## Over-built or cut

- **Separate consent table:** cut. In-row consent fields are sufficient and simpler. No join needed, volume is low, and the fields are small.
- **Event sourcing or CQRS:** cut entirely. This is 10-30 leads per day. A simple activities append log plus the leads table is the full audit trail.
- **Per-stip normalized table:** cut. `stips TEXT[]` on funder_submissions is sufficient. Normalize only if stip-level reporting is needed later.
- **Warmth-scoring in Supabase functions:** cut. n8n writes the score. Moving scoring logic into Postgres adds maintenance surface for no gain.
- **Real-time Supabase subscriptions for the dashboard:** not needed. A shift-start query and a 1-minute n8n poll are sufficient. Real-time adds complexity and a persistent connection.

---

## Top risks

1. **Supabase region wrong at creation.** If the project is created in EU (the default region shown in the Supabase dashboard for non-US browsers), all PII and financial data lands outside the US. There is no migration path; a new project must be created. Verify region before the first row is written.

2. **n8n 48-hour chase job fails silently during work hours.** The chase is a scheduled n8n workflow. If n8n crashes or the VPS reboots between 10:00 and 14:00 ET, document chases are missed and warm deals go cold. Mitigation: Supabase `pg_cron` extension as a backup to email the operator directly if `doc_chase_due < NOW() AND docs_received = FALSE AND stage = 'S3'`, independent of n8n uptime.

3. **DeepSeek receives PII via note-cleanup prompts.** If the operator pastes raw call notes containing the merchant's name, phone, or revenue into a DeepSeek-routed prompt, that data leaves the US and may be retained. The fix is a strict prompt template that strips identifiers before routing to DeepSeek. This is a process risk, not a schema risk, but the schema must not make PII easy to accidentally serialize into a prompt payload.

---

## Constraint flags

- **US data residency:** Supabase project must be US-region. Bank statements must not be stored in Supabase at all; use Supabase Storage (US bucket) with signed URLs expiring in 24 hours, or a US S3 bucket. This gates the build before any data is written.
- **TCPA consent fields:** `consent_timestamp`, `consent_scope`, `tcpa_ip`, and `tcpa_user_agent` are legally required to be accurate and tamper-evident. Do not allow updates to these columns after insert. Enforce with a trigger or RLS policy that blocks UPDATE on these four columns.
- **Budget:** Supabase free tier is sufficient for this volume (500MB database, 1GB storage, 50,000 monthly active users). No paid Supabase tier is needed in the first 90 days. `pgsodium` column encryption is available on the free tier.

---

## Cross-lane flags

- **Page form field mapping to `leads` columns:** the qualification page (front-end or no-code lane) must emit exactly the field names defined here. Any mismatch between page output and schema columns will silently drop data. Coordinate field names before the page is built.
- **n8n DeepSeek prompt template for note cleanup:** the note-cleanup workflow (n8n / agent lane) must strip PII before sending to DeepSeek. The schema cannot enforce this; it must be enforced in the n8n workflow.
- **Funder submission stip tracking:** the sales process lane needs to define what counts as a cleared stip and who marks `stips_cleared = TRUE`. This is a workflow decision, not a schema decision.
- **Bank statement storage and signed URL generation:** the document collection flow (n8n lane) must handle upload routing. The schema only stores a `doc_storage_ref TEXT` field (not designed above; add it to `leads` or `funder_submissions` as needed).
- **pg_cron availability on Supabase free tier:** verify that `pg_cron` is available on free tier before relying on it as the backup chase mechanism. If not, the backup must be a separate always-on process.

---

## Facts to verify

1. Is `pgsodium` column-level encryption available on the Supabase free tier, or only on Pro and above? If not, encryption must be handled in n8n before insert.
2. Does the Supabase free tier support `pg_cron`? This determines whether the backup chase trigger is a Postgres job or a second n8n workflow.
3. Confirm Supabase US-region options at account creation (currently `us-east-1` and `us-west-1`). The dashboard may default to a non-US region based on browser location (Israel).
4. Confirm DeepSeek data retention and residency policy for API inputs. If DeepSeek retains API payloads, any prompt containing merchant PII is a US-residency violation regardless of the model being "cheap."
5. Does the brokerage's funder list define clawback in calendar days or business days, and is `clawback_window_end` best stored as a computed column or set by n8n at funded_date insert?
