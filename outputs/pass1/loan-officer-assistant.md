**Lane:** File completeness, merchant approvability thresholds, and stip-clearing velocity from qualifying call to funded deal.

**Market reality:**

MCA is cash-flow underwriting. Required file: 3-6 months business bank statements (all pages, no gaps), one-page application, government-issued ID, voided check; card-heavy merchants add processing statements. Funders score total deposits, average daily balance, deposit frequency, NSF count, negative days, and existing MCA debits. Floor: $10-15k/month deposits, 6+ months in business, FICO as low as 500-550. Clean file funds in 24-72 hours. Partial statements are the top deal-killer. Heavy stacking or chronic negative days yields declines or outsized rates. TRID, LE/CD, rescission, HMDA do not apply.

**What the market needs:**

The gap between "interested merchant" and "complete file submitted" is where most deals die. The plan treats document-request drafting as a byproduct. It must be a first-class output.

**Opportunities to seize:**

1. Make the document-request draft a mandatory post-call output: exact statement months, account name matching voided check, ID format, 48-hour return deadline. Biggest compression of time-to-funded.
2. Add two pre-screen questions to the qualification page: existing daily or weekly advance payments, approximate negative bank days in the last 90 days. High-risk answers drop the merchant to Tier 2 or Pause before document collection.
3. Build a 48-hour document-chase cadence in Supabase. No response triggers a follow-up with the same list. Primary pipeline leak for a solo operator.
4. Add a funder-submission tracker: funder, submission date, stips, cleared status, offer-expiry. Expired offers and uncleared stips are silent losses the plan cannot surface.

**Hard truths and risks:**

1. Self-reported data is unreliable. A merchant claiming $20k/month can show $9k deposits and two live MCA positions on statements. Confirm deposit range and stacking status on the call before Tier 1 commitment.
2. DeepSeek as default model for workflows touching bank statements or PII is a data-security liability: non-US model, unclear retention. Resolve before document automation goes live.
3. No post-submission tracking exists. Stip lists and offer-expiry windows are where funded deals leak silently.

**Facts to verify:**

- Minimum deposit floor by funder tier in 2026 (reported $5k to $15k+).
- NSF and negative-day tolerance by funder category (reported 0 to 5 days).
- Whether direct funders waive processing statements for low-card-volume merchants.
- US states requiring commercial financing broker disclosures in 2026.

**Cross-lane flags:**

- Compliance: state disclosure obligations attach at document collection and offer presentation; consent must cover multi-funder submission.
- Tech: confirm Supabase is US-region before bank statements or PII enter the system.
- Warming: stacking and negative-day questions belong on the qualification page.
