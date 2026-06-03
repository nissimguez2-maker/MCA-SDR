**Lane:** File completeness, merchant approvability thresholds, and stip-clearing velocity from qualifying call to funded deal.

**Market reality:**

MCA is cash-flow underwriting. Required file: 3-6 months business bank statements (all pages, no gaps), one-page application, government-issued ID, voided check; card-heavy merchants add processing statements. Funders score total deposits, average daily balance, deposit frequency, NSF count, negative days, and existing MCA debits (stacking). Floor: $10-15k/month deposits, 6+ months in business, FICO as low as 500-550. Clean file funds in 24-72 hours. Partial or gapped statements are the top deal-killer. Heavy stacking or chronic negative days yields declines or tiny high-factor offers. TRID, LE/CD, rescission, HMDA do not govern MCA.

**What the market needs:**

The gap between "interested merchant" and "complete file submitted" is where most deals die. The plan treats document-request drafting as an after-call byproduct. It is the primary funded-deal accelerator and must be a first-class immediate output.

**Opportunities to seize:**

1. Promote document-request draft to mandatory post-call output: exact statement months, account name matching voided check, ID format, 48-hour return deadline. Compresses time-to-submitted-file more than any other automation.
2. Add two pre-screen questions to the qualification page: existing daily or weekly advance payments, and approximate negative bank days in the last 90 days. High-risk answers drop the merchant to Tier 2 or Pause before document chase begins.
3. Build a 48-hour document-chase cadence in Supabase. No response triggers a follow-up with the same list. Primary pipeline leak for a solo operator.
4. Add a funder-submission tracker: funder, submission date, stips issued, cleared status, offer-expiry. Expired offers and uncleared stips are silent losses the plan cannot currently surface.

**Hard truths and risks:**

1. Self-reported data is unreliable. A merchant claiming $20k/month and no debt can show $9k deposits and two live MCA positions on actual statements. Confirm deposit range and MCA status verbally before committing Tier 1 time.
2. DeepSeek as default model for workflows touching bank statements or merchant PII is a data-security liability: non-US model, unclear retention, US merchant financial data. Resolve before document automation goes live. Not an open question.
3. No post-submission tracking exists. Stip lists and offer-expiry windows are where funded deals leak silently.

**Facts to verify:**

- Minimum deposit floor by funder tier in 2026 (reported $5k to $15k+).
- NSF and negative-day tolerance by funder category (reported 0 to 5 days).
- Whether direct funders waive processing statements for low-card-volume merchants.
- US states requiring commercial financing disclosures from brokers in 2026.

**Cross-lane flags:**

- Compliance: disclosure obligations attach at document collection and offer presentation; consent must cover submission to multiple funders.
- Tech: confirm Supabase is US-region before bank statements or PII enter the system.
- Warming: stacking and negative-day questions belong on the qualification page.
