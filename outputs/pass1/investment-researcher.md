**Lane:** Segment unit economics from the broker's perspective: which merchant profiles are fundable AND profitable inside a 90-day commission window, clawback exposure, and where to allocate finite selling hours.

---

**Market reality:** ISO/broker commission is 5-15% of funded amount baked into factor-rate markup. Clawbacks on early default are standard at 60-90 days. EV math by segment: high-risk (stacked, under 6 months TIB, sub-$15k/month) averages ~$8k funded at 6% = EV ~$288 after 40% clawback exposure. Mid-tier (12+ months TIB, $20-40k/month, one stack) averages ~$35k at 9% = EV ~$2,520 after 20% default. Sweet-spot (18+ months TIB, $40k+/month, no stack) averages ~$70k at 10% = EV ~$6,160 after 12% default. That is a 20x EV gap per worked lead. Cold-to-funded conversion on raw aged leads: 2-3%; screened warm inbound: 8-12%. The $18-22B market figure likely blends revenue-based financing and factoring; ISO-originated broker volume is materially smaller.

**What the market needs:** Brokers screening for clawback survivability, not just approvability. Funders price factor rates to absorb their default exposure. Brokers absorb clawbacks with no such offset.

---

**Opportunities to seize:**

1. **Target 18+ months TIB, $30-50k/month, no stack, card-heavy operators (restaurants, medical/dental, auto repair, salons).** Approval 60-75%, default 12-15%, EV $5,000-$8,000. Highest hourly yield in the 90-day window.

2. **Hard-gate stacked and sub-$15k/month merchants at the qualification page before any call.** UCC or stacking flag at intake eliminates the 40%-default segment without burning a prime-hour dial.

3. **Re-engage the existing funded merchant book first.** A merchant who funded, repaid, and is 6+ months post-payoff is the highest-conversion, lowest-default lead type with near-zero sourcing cost.

4. **Filter on declared funding need before any human contact.** Sub-$15k stated need signals $400-700 gross commission; deprioritize immediately.

---

**Hard truths and risks:**

1. **Clawback risk is asymmetric and unaddressed.** One $70k advance defaulting in week 8 wipes commission on two other funded deals. The plan has no segment-gating by default probability and no funded-date field in the Supabase schema.

2. **Aged-lead vendors route the most toxic files.** The same recycled high-risk lead goes to 5-10 brokers simultaneously. Vendor-aged default rates likely exceed 30-40%. The plan lists vendors as a legitimate channel with no caveat.

3. **Tier 3 nurture is inviable on a 90-day window.** Any lead needing 30+ days to fund cannot survive the clawback window. The Tier 3 path pays out after the run ends.

---

**Facts to verify:**

- Clawback window by funder: 60 vs. 90 vs. deal-specific. Most important number in the 90-day commission model.
- ISO commission ceiling per funder: some cap at 10% regardless of broker pricing. Confirm against actual ISO agreements.
- Default rate by industry: restaurants vs. trucking vs. construction differ materially from the blended 15-20% headline.
- $18-22B scope: confirm whether the source includes revenue-based financing and factoring.

---

**Cross-lane flags:**

- Compliance: NY and CA commercial financing disclosure laws affect qualification page copy and consent capture.
- Tech/ops: Supabase schema needs funded-date and clawback-window fields from day one; retrofitting loses commission-survival tracking on early deals.
- Outbound strategy: Raw UCC lists need a stack-check filter before Tier 1 assignment or the highest-default segment tops the daily queue.
