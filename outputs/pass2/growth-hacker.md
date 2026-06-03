# Pass 2 Audit: Growth Hacker Lens

**Lane:** Outbound-to-page funnel design and lean experiments a solo operator can actually read and act on.

---

## Strategy

The full funnel runs: signal-sourced touch -> single SMS or email with a transactional hook -> qualification page -> warmth score -> booking link (high score) or nurture drip (low score) -> call -> file submitted -> funded -> 60-90 day clawback cleared. Every experiment lives inside this chain. The North Star metric is **funded deals per month**, not conversations or page visits. The page is a downstream warming and consent surface, not a traffic channel; its job is to disqualify the unbankable before they eat call time and to capture TCPA one-to-one consent before any SMS follow-up. At 10-30 daily touches, no A/B split reaches statistical power; the only honest learning model is sequential and directional: run version A for two weeks, record outcomes, switch to B for two weeks, compare directionally.

---

## Tactics

**1. Define the funnel-stage metric set now, before anything is built.**

North Star: funded deals per month (revenue proxy).
Funnel metrics worth tracking, in order:
- Page completion rate (form submitted / link opened).
- Qualification rate (scored above threshold / form submitted).
- Booking rate (call booked / qualified).
- Show rate (call completed / call booked).
- Funded rate (funded / call completed).

Add a clawback survival rate at 90 days as a lagging quality check. That is six numbers. Track them weekly in a single Supabase view. No dashboards beyond this until month two.

**2. Build the qualification page around transactional urgency language first; calculator is secondary.**

The page headline should mirror the trigger that got the click. A re-engagement touch says "Your funding profile from [date] is still on file. See what you qualify for now." The page headline echoes that: "Check your current approval range in 2 minutes." The payback calculator appears on step two, after the merchant has committed revenue and need; it is a completion driver, not a hook. Hard-gate on: time in business under 6 months, monthly revenue below 8k, active judgment or bankruptcy, already in default on an advance. The gate kills a call that would not fund; every killed call is call time recovered. Consent capture (TCPA one-to-one, SMS and call permission, funder name disclosed) is a required field before the booking link appears. Without it, do not text.

**3. Run sequential opener experiments, not simultaneous splits.**

Two weeks each, one variable at a time. Experiment 1 (weeks 1-2): trigger-referencing opener ("Your last advance was with [funder from UCC]. They typically re-up at 60 percent payoff. You may be at that point.") vs generic re-engagement opener. Measure: reply rate and page completion rate. Experiment 2 (weeks 3-4): SMS with consent vs email only. Measure: open-to-page rate. Experiment 3 (weeks 5-6): page with calculator vs page without. Measure: completion rate and qualification rate. Log results in a plain table. Call a winner if the directional gap is large enough that you would confidently bet on it. Accept that n is too small for p-values; you are calibrating intuition, not proving significance.

**4. Score warmth on concrete behaviors, not inferred intent.**

Four signals, each with a point value assigned once and not changed without a reason:
- Form completed (all required fields): 30 points.
- Real phone entered (passes format check): 15 points.
- Callback time selected: 20 points.
- Call booked via calendar link: 35 points.

A score above 50 routes to the booking link immediately. A score of 30 to 49 triggers an automated SMS (consent required) or email nudge within 15 minutes. A score below 30 drops to Tier 3 nurture. This is not AI scoring; it is a deterministic rule. No model call needed. Build it in n8n in two hours.

**5. SMS follow-up is the single highest-leverage experiment IF consent is clean.**

The market brief confirms warm re-engagement converts several times better than cold. SMS open rates are materially higher than email. The constraint is consent: UCC-sourced contacts do not arrive with TCPA one-to-one consent; texting them before they opt in on the page is non-compliant under the January 2025 FCC rule. The experiment is: for contacts who complete the page and grant SMS consent, send a 60-word SMS within 10 minutes of form submission. Compare show rate on booked calls for SMS-nudged vs email-only. This is the one test where a directional result at low volume is still actionable, because the cost of the SMS channel itself is the variable being tested, not a marginal copy tweak.

---

## What to avoid

- Tracking page views, click-through rates, or open rates as primary metrics. They measure activity, not progress toward funded.
- Running A/B tests simultaneously at 15 contacts per day. You need 200-plus conversions per variant for a meaningful split; that is months away. Run sequential tests and accept directional evidence.
- Adding analytics tools (Mixpanel, Amplitude, Heap) in the first 60 days. Supabase event logs and a spreadsheet are sufficient.
- Optimizing the page for organic search. It has no organic traffic; SEO work here is waste.
- Scoring warmth with a model call on every form submission. Rules are cheaper, faster, and just as accurate at this volume.

---

## Over-built or cut

- The three-agent architecture in the plan is over-specified for this lane. The qualification page, warmth scoring, and booking routing are deterministic rule logic, not reasoning tasks. A single n8n workflow handles form webhook -> score calculation -> routing -> CRM write -> nudge send. No dedicated "Warm Lead Engine" agent is needed until volume exceeds what a rule can handle.
- The "behavioral signals become warmth score" framing in the plan is correct but risks being over-engineered. Four hard-coded point values beat a trained model at this volume. Defer model-based scoring.

---

## Top risks

1. **Consent gap kills the SMS channel.** If the page is not live and capturing TCPA one-to-one consent before outreach begins, every text sent to a UCC-sourced contact is a liability. The page must be live and compliant before the first SMS is sent; the SMS experiment cannot run until then.

2. **Low volume makes funnel metrics noisy for 30-plus days.** With 10-30 daily touches and a multi-step funnel, weeks 1-2 will produce fewer than 20 page visits. Completion rate calculated on 8 observations is noise. Resist the urge to optimize early; let data accumulate for 3-4 weeks before acting on funnel-stage numbers.

3. **The booking link creates a false confidence signal.** A booked call is the warmest exit from the page, but show rates for cold-to-page-to-book flows in financial services are often below 50 percent. Tracking booked calls without show rate overstates pipeline. Track show rate from day one; a booked call that does not show is not a funded deal.

---

## Cross-lane flags

- The compliance posture on TCPA one-to-one consent and state broker registration (market-brief.md "Facts to verify") must be resolved before the page goes live and before SMS is used. This is a compliance lane item, not a growth hacker item; flagging it because it is the binding constraint on tactic 5.
- UCC-1 sourcing (opportunity 4) feeds this funnel's cold re-engagement arm. The trigger-referencing opener in tactic 3 depends on clean funder and payoff-date data from UCC filings. If that data is dirty or delayed, the opener reverts to a generic hook and conversion drops. Sourcing quality is a cross-lane dependency.
- Call-queue ranking (opportunity 5) determines which page-completed leads get called first. The warmth score from the page should feed directly into the queue-ranking logic. This is a Sales Intelligence lane item; growth hacker lane delivers the score, that lane consumes it.
- Document collection (opportunity 3) is fully downstream of this funnel and is not a growth hacker item. Flagged only because a funded rate below expectation can be caused by doc-collection failure, not funnel failure; misattributing that to page conversion is a diagnostic risk.

---

## Facts to verify

- TCPA one-to-one consent rule (FCC January 2025): does it apply to business-to-business SMS to sole proprietors and small LLCs, or only to consumer numbers? The court challenge status as of mid-2026.
- SMS open rates for financial services re-engagement vs email: the "materially higher" claim is well-established generally but verify for the MCA / small-business lending segment specifically.
- Show rates for calendar-booked calls originating from a cold outbound-to-page flow in MCA brokerage: industry benchmarks, if any exist.
- Cal.com free tier limits for a solo operator: booking volume cap and webhook support at the free tier.
- Whether a plain Supabase webhook + n8n free tier can handle the form-to-score-to-nudge flow within the 10-minute window assumed in tactic 5, at the planned daily volume, without hitting n8n execution limits on the free plan.
