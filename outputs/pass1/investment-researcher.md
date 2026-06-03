**Lane:** Segment unit economics: fundable AND profitable merchant profiles inside a 90-day commission window, clawback exposure, and solo broker hour allocation.

---

**Market reality:** ISO/broker commission is 5-15% of funded amount baked into factor-rate markup. Clawbacks on early default are standard at 60-90 days. EV per funded deal: high-risk (stacked, under 6 months TIB, sub-$15k/month) = ~$288 after 40% clawback. Mid-tier (12+ months TIB, $20-40k/month, one stack) = ~$2,520 after 20% default. Sweet-spot (18+ months TIB, $40k+/month, no stack) = ~$6,160 after 12% default. A 20x EV gap per worked lead. Cold-to-funded conversion: 2-3% on raw aged leads; 8-12% on screened warm inbound. The $18-22B market figure likely blends revenue-based financing and factoring; ISO-originated broker volume is materially smaller.

**What the market needs:** Brokers screening for clawback survivability, not just approvability. Funders price factor rates to absorb their default. Brokers absorb clawbacks with no offset.

---

**Opportunities to seize:**

1. **Target 18+ months TIB, $30-50k/month, no stack (restaurants, medical/dental, auto repair, salons).** Approval 60-75%, default 12-15%, EV $5,000-$8,000. Best hourly yield in the 90-day window.

2. **Hard-gate stacked and sub-$15k/month merchants at the qualification page.** UCC or stacking flag at intake eliminates 40%-default leads before any call.

3. **Re-engage the existing funded book first.** Merchant who funded and repaid, 6+ months post-payoff: highest conversion, lowest default, near-zero sourcing cost.

4. **Filter on declared funding need before human contact.** Sub-$15k stated need signals $400-700 gross; deprioritize immediately.

---

**Hard truths and risks:**

1. **Clawback risk is asymmetric and unaddressed.** One $70k advance defaulting in week 8 wipes commission on two other funded deals. No default-probability gating exists. No funded-date field in the Supabase schema.

2. **Aged-lead vendors route toxic files.** The same recycled lead goes to 5-10 brokers simultaneously. Vendor-aged default rates likely exceed 30-40%. The plan lists vendors as legitimate with no caveat.

3. **Tier 3 nurture is inviable on a 90-day window.** Any lead needing 30+ days to fund cannot clear the clawback window.

---

**Facts to verify:**

- Clawback window by funder: 60 vs. 90 vs. deal-specific. Most important number in this model.
- ISO commission ceiling: some funders cap at 10%. Confirm against actual ISO agreements.
- Default rate by industry: restaurants vs. trucking vs. construction differ from the 15-20% blended figure.
- $18-22B scope: confirm whether source includes revenue-based financing and factoring.

---

**Cross-lane flags:**

- Compliance: NY and CA commercial financing disclosure laws affect qualification page copy and consent capture.
- Tech/ops: Supabase needs funded-date and clawback-window fields from day one; retrofitting loses tracking on early deals.
- Outbound: Raw UCC lists without a stack-check filter send the highest-default segment to the top of the queue.
