# Pass 2 Audit: Behavioral Nudge Engine Lane

**Lane:** Qualification page design, progressive field order, calculator value exchange, warmth scoring, hard gating, and booking routing.

---

**Strategy:** The page is not a form; it is a low-friction, high-signal sales instrument that does four jobs simultaneously: mirrors the merchant's urgent transactional language to reduce bounce, extracts qualification data as a series of micro-commitments, computes real payback cost to build trust and disqualify the math-blind, and routes high-signal completers straight to a booking link before they cool off. Completion and booking lift are the output metrics; hard-gating bad-fit and distress merchants before a call is the cost-control mechanism. Every field ordering and UX choice is justified by cognitive load theory and commitment-consistency psychology, not aesthetics.

---

**Tactics** (ranked by leverage)

**1. Hard-gate the distress merchant at the front, not the back.**
Field 3 or 4, after business name and phone, insert the single disqualifier: "Do you currently have an MCA advance you are unable to repay?" with a Yes/No toggle. A "Yes" response does not submit to the SDR queue; it routes to a static page: "We are not able to help with MCA restructuring or consolidation at this time. For debt counseling, contact [SBDC link]." This gate must fire before any warmth score accumulates and before any consent checkbox is shown. Showing the gate early removes the false hope problem and protects against distressed-merchant clawback chains from day one. Do not ask this as the first question; business name and phone first so partial data is captured even on exit.

**2. Order fields as ordered micro-commitments, lowest friction first, highest commitment last.**
Exact sequence and logic:
- Step 1: Business name (zero threat, pure identity, anchors commitment to a real entity).
- Step 2: First name, best phone number (now they have given an identity plus a contact; consistency bias pulls them forward).
- Step 3: Distress gate (explained above; if Yes, exit).
- Step 4: Industry (single dropdown; 10 choices; auto-suggest; sets risk-scoring context).
- Step 5: Monthly revenue (three radio buckets: Under 15k / 15k-50k / 50k-plus; not a text field; removes friction and anchors to realistic bands).
- Step 6: Time in business (Under 6 months / 6-24 months / Over 24 months; same pattern).
- Step 7: Funding amount needed (radio: Under 20k / 20k-75k / 75k-plus).
- Step 8: Urgency (radio: This week / Within 30 days / Just exploring; feeds score directly, flags payroll-critical visitors in copy).
- Step 9: Use of funds (short free-text, 80 char max, no placeholder text; urgency keywords like "payroll," "inventory," "open location" are parsed by scoring logic).
- Step 10: Existing debt load (radio: None / Under 10k / 10k-50k / Over 50k; stacking flag).
- Step 11: Consent checkbox (TCPA one-to-one language; non-pre-checked; required to advance).

Total: 11 data points across a multi-step form with a visible progress indicator (Step 2 of 5, where steps group the fields; not 11 individual steps). Each "page" of the multi-step form ends with a forward button, which is itself a micro-commitment. Never show all 11 fields on one scroll.

**3. MCA payback calculator as the value exchange and trust signal, not a gimmick.**
After Step 7 (funding amount) and before Step 8, surface an inline calculator: "Here is what your advance costs." Inputs auto-fill from prior answers. Show: Advance amount, factor rate range (1.20-1.49 depending on revenue and time in business), total payback, estimated daily payment at 21 days/month, and a plain-language note: "Factor rate is not an interest rate. Your total payback is fixed at the time of offer." This is the market-brief's "transparent factor-rate-is-not-APR" requirement in action. A merchant who completes the calculator has already modeled their payback, which means they self-select for math literacy and seriousness. The calculator interaction itself is a warmth signal (+10 score points if the merchant adjusts the sliders, meaning they engaged). Do not show APR equivalents; the market-brief flags this as a confusion vector.

**4. Warmth scoring with explicit thresholds and routing logic.**
Score starts at 0. Add points on each signal:

| Signal | Points |
|---|---|
| Form fully completed | +20 |
| Real phone number format (10-digit US, passes regex) | +15 |
| Urgency = "This week" | +20 |
| Urgency = "Within 30 days" | +10 |
| Use-of-funds free text contains payroll/urgent/this week/open/inventory | +10 |
| Funding amount 20k-75k | +10 |
| Funding amount 75k-plus | +15 |
| Time in business over 24 months | +10 |
| Monthly revenue 50k-plus | +15 |
| Monthly revenue 15k-50k | +5 |
| Existing debt None | +10 |
| Existing debt under 10k | +5 |
| Existing debt over 50k | -15 (stacking flag) |
| Calculator sliders adjusted (engagement signal) | +10 |
| Callback requested explicitly | +15 |

Routing thresholds:
- Score 80+: "You qualify for a fast review. Book a 15-minute call now." Route to Cal.com or Calendly embed on the same page. Do not make them navigate away.
- Score 50-79: "A funding specialist will call you within one business day." Submit to SDR queue as Tier 1 or Tier 2 depending on urgency.
- Score under 50: "We will review your information and reach out if there is a good fit." Submit to nurture queue; no booking link offered; SDR gets a Tier 3 flag.

**5. Consent copy that is TCPA-compliant and specific.**
The checkbox label must be non-pre-checked and must state the specific requester, the specific channels, and the specific purpose. Example language: "By checking this box, I consent to be contacted by [Brokerage Legal Name] at the phone number I provided above, including by autodialer or text message, regarding business funding options. I understand this consent is not required to obtain services. Message and data rates may apply." The consent language must name the brokerage, not a generic "our partners." January 2025 FCC one-to-one consent rule means a blanket "partners may contact you" clause is non-compliant. This is a legal hard constraint, not a UX preference.

---

**What to avoid**

- **Fake scarcity:** No "Only 3 spots left this week" or countdown timers. MCA advances are not inventory-constrained; this claim is false and legally risky.
- **Pre-checked consent boxes:** Illegal under TCPA one-to-one consent rules; do not do this even if it lifts completion rates.
- **Factor-rate-as-APR framing:** Do not convert factor rate to APR in the calculator or anywhere on the page. The conversion is technically possible but creates an inflated-cost impression that either scares off the merchant or, if the merchant later learns the comparison was apples-to-oranges, destroys trust. The plan's own compliance posture bans this.
- **Social proof fabrication:** No invented testimonials or fake "Funded 47 businesses this week" counters. Genuine trust markers only: real funder names where allowed, actual disclosures.
- **Aggressive re-targeting pop-ups on exit:** An exit-intent modal that says "Wait! Don't miss out!" is a dark pattern that signals desperation and is inconsistent with the transactional, businesslike tone the merchant expects.
- **Hiding the distress gate:** Do not bury the "are you in default" question at the end of the form after the merchant has invested 10 minutes. That is a manipulative waste of their time and the SDR's.
- **Ambiguous "learn more" or soft CTAs at the booking step:** At score 80+, the CTA must be direct: "Book your call now." Soft language at high intent moments costs booked calls.

---

**Over-built or cut**

The plan proposes a "funding-readiness check" as a separate deliverable from the calculator. These are the same surface. One inline tool that shows payback math and flags whether the merchant meets baseline criteria (revenue, time in business) covers both. Build one, not two. The plan also mentions "callback requested" as a warmth signal but does not specify how it is captured; a single "Request a callback instead" toggle at the booking step, scored at +15, is sufficient. No separate callback form or flow needed.

---

**Top risks**

1. **TCPA consent language is legally wrong on launch.** If the consent checkbox names the wrong entity, uses pre-1-to-1 "partner" language, or is pre-checked, every contact made through the page is potentially non-compliant. This must be reviewed by a US telecom compliance attorney before any outreach begins, not after. The Jan 2025 FCC rule is under court challenge but treat it as live until overturned.

2. **Distress-merchant gate is bypassed by a false "No" answer.** A merchant in MCA default who clicks "No" to the distress question proceeds through the form. The scoring model's stacking flag (existing debt over 50k, minus 15 points) and urgency mismatch will often catch this, but not always. Downstream, the SDR must verify existing debt on the call and kill the deal early if default is confirmed. The gate reduces, not eliminates, this risk.

3. **Cal.com or Calendly booking embed fails silently during US hours.** The operator is outside US hours during the build window. If the booking embed breaks, high-score leads hit a dead end and convert to zero. The page must have a fallback: if the embed fails to load within 3 seconds, show a plain "Submit your info and we will call you" form instead. This is the always-on constraint applied to the most valuable exit point.

---

**Cross-lane flags**

- The warmth score this lane produces is the primary input to the call-queue ranking system (Opportunity 5, ranking lane). Score schema must be agreed between these two lanes before either is built; otherwise the scoring table above may be rebuilt twice.
- The consent checkbox capture feeds the document collection engine (Opportunity 3); the legal consent language and the timestamp of consent should be stored in Supabase at the same time as the lead record, not as a separate write.
- The distress-merchant exit page with an SBDC link is content, not engineering. It is a one-time write. Flag to whoever owns copy.
- TCPA one-to-one consent for UCC-sourced contacts (Opportunity 4) is a related but distinct consent scenario. The page consent language does not cover cold-sourced contacts. Flag to compliance lane.

---

**Facts to verify**

- Jan 2025 FCC one-to-one TCPA consent rule: current legal status under Eleventh Circuit challenge and whether prior consent forms need to be updated retroactively.
- Whether Cal.com free tier supports iframe embed with custom redirect on submission, or whether Calendly is required for that routing.
- Whether a "Do you have an MCA you cannot repay" gate question, standing alone, creates any duty-to-counsel or referral obligation under state law (NY, CA, TX at minimum).
- Supabase free tier row-level security: confirm that partial form submissions (steps 1-3 before distress gate exit) are stored with a "disqualified" flag and not surfaced to the SDR queue, and that Supabase US-region is selected to avoid EU data residency issues with US PII.
- Whether the inline factor-rate calculator output constitutes a "commercial financing disclosure" triggering CA SB 1235 or NY CFDL disclosure requirements, even before a formal offer is made.
