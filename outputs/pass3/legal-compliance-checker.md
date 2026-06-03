# Pass 3 Audit: Legal Compliance Checker

**Lane:** TCPA, DNC, state commercial financing disclosure, broker registration, and consent-capture build for the MCA SDR qualification page and outreach cadences.

---

## Implementing the strategy

### Tactic 1: Cold UCC outreach by phone, voicemail, and email

**Phone (live agent, manual dial):** Calling a business's dedicated business landline is generally exempt from the federal National DNC Registry's B2B carve-out (TSR Section 310.6(b)(7) exempts calls to businesses except nondurable office/cleaning supply solicitations). However, verify with counsel: if the UCC debtor's contact number is a sole proprietor's personal cell or a mixed-use cell, courts have found TCPA wireless restrictions apply regardless of business context. A Ninth Circuit ruling created a presumption that mixed-use numbers are residential for DNC standing purposes. UCC pulls almost always surface mobile numbers. Treat every UCC-sourced cell as a potential TCPA-covered number.

**Email:** CAN-SPAM applies. Required: physical address, opt-out mechanism, truthful subject line, no deceptive headers. Email to a business is not exempt from CAN-SPAM. This is the lowest-risk cold channel for UCC outreach. Proceed, with CAN-SPAM mechanics in every message.

**Voicemail (live or ringless):** The FCC ruled in 2022 (FCC 22-85) that ringless voicemail delivered to a wireless phone is a "call" using an artificial or prerecorded voice under TCPA. That ruling stands as of mid-2026. One human-recorded voicemail per cadence using a live dial (not a drop service) is the only defensible approach without prior written consent. Do not use any ringless voicemail drop service for cold UCC contacts. Pushback: the plan references "one voicemail per cadence" without specifying delivery method. Nail this down before launch.

**Autodialer or sequential dialer for UCC cold contacts:** Do not do this. TCPA requires prior express written consent for autodialed calls or texts to wireless numbers, B2B or not. UCC-sourced contacts have no page opt-in. Using an ATDS on cold UCC numbers is a direct TCPA violation. Live manual dialing only for cold UCC pool.

### Tactic 2: Qualification page consent checkbox and payback calculator

**Consent checkbox:** This is the fulcrum of the whole system. Without a valid, attorney-reviewed consent checkbox, SMS is blocked entirely and the TCPA exposure on any subsequent automated outreach is uninsured.

The checkbox must be built as described below in the Build spec. The one-to-one FCC consent rule was vacated by the Eleventh Circuit in January 2025 and subsequently repealed by the FCC (September 2025). The current standard reverts to the pre-2025 FCC rules: prior express written consent requires clear and conspicuous disclosure, identification of the seller, description of contact methods including autodialer and prerecorded voice, and a non-pre-checked affirmative act. The Fifth Circuit separately rejected the "prior express written consent" rule expansion in March 2026 (TCPA Reset, Holland and Knight, March 2026). Verify with counsel which circuit's interpretation governs for a given merchant's location before relying on any particular consent formulation.

**Payback calculator:** California SB 362, effective January 1, 2026, prohibits using "rate" or "interest" in a way that could mislead a recipient about APR for commercial financing of $500,000 or less, and requires re-disclosure of APR every time a charge, pricing metric, or financing amount is stated during an application process. The inline calculator states a factor rate and total payback. If California merchants are in scope (they are, UCC pulls are nationwide), verify with counsel whether the calculator output constitutes a "statement of a charge or pricing metric" that triggers mandatory APR re-disclosure under SB 362 or DFPI enforcement posture. The plan's language ("factor rate determines total payback and is not APR") directly conflicts with SB 362's prohibition on stating a factor rate without also stating APR. This is a hard stop for CA merchants until counsel confirms.

New York CFDL (Article 8 NY Financial Services Law, regulations effective 2023) requires standardized disclosures at the time a financing offer is extended, and applies to brokers as well as lenders for transactions up to $2.5M. Verify with counsel whether the calculator output, which produces a total payback figure, constitutes an "offer" under NY CFDL triggering disclosure obligations before a formal underwritten offer is made.

### Tactic 5: Auto-enforced call window

**Build it.** TCPA prohibits calls before 8:00 AM or after 9:00 PM at the recipient's local time. The operator works from Israel (17:00-02:00 Israel time), making time-zone errors a near-certain manual risk. Time-of-day TCPA cases surged in 2025 (over 507 class actions in Q1 2025 alone, up 112% year-over-year per search results). The n8n call-window enforcement must use the merchant's area code or address to determine local time, block queue population outside those hours, and log the basis for the time-zone determination. This is not optional. It is among the highest-probability failure modes for a solo operator in a foreign time zone.

### SMS (post-consent only, per parked tactic)

The plan correctly parks SMS until after page consent. Do not deviate. Autodialed or pre-templated texts to any number, even a business cell, require prior express written consent. Post-consent, one touch only per the strategy. Verify with counsel whether the consent language on the page covers SMS specifically (it must name text messages as a channel).

---

## Build spec

### Consent checkbox copy (attorney must approve final language before any page goes live)

The checkbox must be:
- Non-pre-checked, requiring an affirmative click
- Positioned immediately adjacent to the submit or "book call" button, not buried in a footer
- In font size equal to or larger than surrounding body text

Required disclosure elements in the checkbox label (all of these, not a subset):

1. Named brokerage: full legal business name of the brokerage, not a DBA alone
2. Channel enumeration: "calls (including using an autodialer or prerecorded voice), text messages, and email"
3. Purpose: "regarding business financing options"
4. Revocation: "Consent is not required to receive services. You may revoke consent at any time."
5. Phone number field reference: "at the phone number and email provided above"

Example draft (not final, counsel must approve): "By clicking, I authorize [Legal Brokerage Name] to contact me at the phone number and email above using calls (including using an autodialer or prerecorded voice), text messages, and email regarding business financing options. Consent is not required to receive services or qualify for financing. I may revoke consent at any time."

### Consent record-keeping (per record, per lead)

Each consent event must log and store in Supabase:
- Timestamp (UTC, millisecond precision)
- IP address of the submitting device
- Full verbatim checkbox copy as displayed at time of submission (not a version reference, the literal text)
- Page URL and form version identifier
- Phone number and email as entered by the user
- Area code or state derived for call-window enforcement

Retention: minimum 4 years (TCPA statute of limitations is 4 years). Store as an immutable append-only row; do not overwrite.

### DNC scrubbing

For UCC cold calls (no page opt-in):
- Scrub against the federal National DNC Registry before any dial. Register with the FTC's Subscription System (first 5 area codes free; $82/area code/year beyond that as of October 2025).
- Scrub against state DNC registries for the 12 states that maintain them. Identify which states are in the UCC pull and subscribe accordingly.
- For the prior funded book: established business relationship (EBR) exemption may apply for 18 months from the last transaction date, but verify with counsel whether MCA fundings qualify as an EBR for the specific state rules, and whether the exemption covers cell phones under TCPA.
- Log every scrub: date of scrub, registry version date, result (cleared or suppressed). Do not call suppressed numbers under any circumstances.

### Call-window enforcement

In n8n, before any lead enters the active call queue or any automated touch is triggered:
1. Resolve merchant local time zone from area code (use a free lookup API such as numverify or similar) with address state as a fallback.
2. Check whether current UTC time falls between 08:00 and 21:00 in the merchant's local zone.
3. If outside window: hold the record, re-check at the next queue population run.
4. Log the zone used and the window check result per touch attempt.
5. Hard block: no automated SMS, no triggered email-to-call handoff, outside the window.

### Record-keeping (full compliance set)

- Consent records: Supabase, immutable, 4-year retention
- DNC scrub logs: date, registry version, result, per number per scrub run
- Call logs: timestamp, duration, outcome, caller ID used, merchant local time zone confirmed
- Opt-out log: any revocation received by any channel, timestamp, action taken within 10 business days (TCPA opt-out rules effective April 11, 2025 require honoring revocation within 10 business days by any reasonable means)
- State disclosure delivery log: if operating in CA, NY, TX, CT, FL, GA, KS, MO, VA, UT, record when and which disclosure was provided

### "Do not launch until" checklist

- [ ] Attorney has reviewed and approved final consent checkbox copy in writing
- [ ] Brokerage is registered (or verified exempt) in each state where UCC outreach will occur, specifically: TX (OCCC registration, deadline December 31, 2026, but engagement before registration is restricted), CT (Banking Commissioner, required before October 1, 2024, meaning NOW), NY (CFDL broker registration if applicable), CA (DFPI, verify CFL broker license requirement), and any other disclosure-law state where outreach begins
- [ ] CA SB 362 APR re-disclosure obligation confirmed not triggered by calculator, or calculator revised to include APR output for CA merchants
- [ ] Federal DNC Registry subscription active and first scrub completed before any UCC dial
- [ ] State DNC scrubs completed for all states in the UCC pull
- [ ] Supabase consent-log schema deployed with immutable rows and confirmed US-region storage
- [ ] n8n call-window enforcement live and tested with Israel-timezone offset edge cases (midnight ET crossover, DST transitions)
- [ ] EBR exemption scope confirmed with counsel for prior funded book (cell phones, state-specific)
- [ ] Voicemail delivery method confirmed as live-agent dial only (no ringless drop service)
- [ ] SMS suppressed in all automated flows until consent record confirmed in Supabase
- [ ] Opt-out honoring procedure documented: any "stop," "unsubscribe," or verbal revocation logged and acted on within 10 business days
- [ ] CAN-SPAM mechanics in all outbound email templates: physical address, opt-out link, truthful subject

---

## Over-built or cut

**Over-built:** Do not build a separate compliance-monitoring agent or real-time regulatory-alert system. At this volume and budget, a manual quarterly review of the fact-to-verify list below is sufficient. Automation for compliance monitoring is premature and introduces false confidence.

**Cut:** Do not use any third-party lead-verification or consent-append vendor that claims to "cover" TCPA liability via their own consent flow. The brokerage is the "seller" on the hook; only a consent event on the brokerage's own page creates defensible prior express written consent for the brokerage.

**Cut:** Do not run the payback calculator for CA merchants without counsel confirmation on SB 362. Either gate CA merchants out of the calculator output, add an APR field, or get written counsel sign-off. This is not a judgment call the operator makes alone.

---

## Top risks

**Risk 1: UCC cold cell-dial without ATDS produces TCPA exposure for sole proprietors.** The plan assumes UCC-sourced merchant numbers can be cold-called by live agent without consent. For most business landlines this is defensible. But UCC filings for sole proprietors and single-member LLCs frequently surface the owner's personal cell. Manual live-agent dialing to a cell is not per se a TCPA violation, but if the owner later claims the number is residential or mixed-use, TCPA liability attaches. At $500 to $1,500 per violation (and class action exposure), a batch of 100 misclassified cells is a $50K-$150K exposure. Mitigation: run all UCC numbers through a wireless vs. landline lookup before dialing; flag cells for heightened scrutiny; do not autodial any cell without consent.

**Risk 2: State disclosure law registration gap.** The brokerage contacts merchants in at least 10 disclosure-law states. TX requires OCCC registration (grace period to December 31, 2026, but verify whether outreach before registration triggers the "engaged in business" definition). CT required registration by October 1, 2024, meaning any CT outreach now without registration is potentially an ongoing violation. CA requires DFPI licensing for commercial financing providers; the broker-exemption scope is not clear for pure brokers under SB 1235 and SB 362 combined. Operating across these states without confirmed registration status is the single highest-probability regulatory enforcement risk in this plan.

**Risk 3: Call-window failure from Israel time zone.** The operator's shift ends at 02:00 Israel time, which is 19:00 ET in winter and 18:00 ET in summer. But automated follow-up triggers (document chase, booking nudges) fire on n8n schedules that could be set in UTC or Israel time by mistake. A single misconfigured cron job can send texts or trigger calls at 6:00 AM or 10:30 PM merchant local time. Given 2025's 112% surge in TCPA time-of-day class actions, this is not a theoretical risk. The n8n enforcement logic must be audited against daylight saving time transitions for both Israel and US time zones, which do not change on the same calendar dates.

---

## Constraint flags

These are hard gates before any outreach of any kind:

1. **Brokerage registration status in each outreach state must be confirmed or obtained.** CT is already past deadline. TX deadline is December 31, 2026, but "engaged in business" triggers may apply before registration. Do not contact merchants in a state where the brokerage is not registered or confirmed exempt until counsel clears it.

2. **Attorney sign-off on consent checkbox copy.** No SMS ever. No ATDS calls ever. Without this, the entire gating mechanism is void.

3. **CA SB 362 APR obligation resolved for the calculator.** Effective January 1, 2026. This is already in effect. Any prior CA merchant contact using the current factor-rate-only calculator language may already be exposed.

4. **DNC scrub completed before first UCC dial.** Not concurrent. Before.

---

## Cross-lane flags

- **Data Engineer / Backend Architect:** Supabase consent log must be append-only (no UPDATE or DELETE on consent rows). Column-level encryption on phone, email, and IP fields. Confirm US-region storage before any PII is written. This is a data-architecture requirement driven by compliance.
- **Automation Governance Architect:** n8n workflow for document chase and booking nudges must inherit the call-window enforcement check. Any workflow that sends a message to a merchant must pass through the same time-zone gate, not just the call queue.
- **Sales Outreach lens:** The "5 touches over 14 days" UCC cadence must be scoped to cleared states only, live-agent dial only on cells, and human voicemail only. The cadence design cannot assume autodialing or ringless drop.
- **Growth Hacker / page build:** The calculator must be toggled by merchant state. CA merchants must not receive a factor-rate-only output until SB 362 APR obligation is resolved.

---

## Facts to verify

Counsel must confirm or deny each of these before launch. These are claims grounded in search results, not settled legal conclusions.

1. **TCPA and UCC sole-proprietor cell calls:** The Eleventh Circuit (January 2025) vacated the FCC one-to-one consent rule; the FCC repealed it (September 2025). The current standard reverts to pre-2025 "prior express consent" with its plain meaning. Verify: does a live-agent manual dial to a sole proprietor's cell number obtained from a UCC filing require prior express written consent, or is ordinary express consent (oral or implied) sufficient for non-autodialed calls? The answer governs whether cold cell dials from the UCC pool are permissible before page opt-in. Source: Wiley Law (January 2025), FCC repeal (September 2025), Fifth Circuit TCPA Reset (March 2026).

2. **TCPA call-consent for B2B cells, mixed-use numbers:** The Ninth Circuit presumption on mixed-use numbers (residential until proven otherwise) creates standing risk for any cell dial regardless of business intent. Verify whether this presumption applies in the circuits covering TX, FL, NY, CA (key UCC states). Source: PossibleNOW DNC B2B analysis, Ninth Circuit caselaw.

3. **Ringless voicemail: confirmed TCPA call.** FCC 22-85 (November 2022) rules it a call using artificial or prerecorded voice. Status confirmed through 2026 per search results. Verify: does using a live-agent press-1-to-leave-voicemail system (not a drop service) change the analysis? Counsel must draw the line between a drop service and an agent-initiated voicemail.

4. **State DNC registries for UCC pull states:** Twelve states maintain state registries not subsumed by federal DNC. Identify which of the planned outreach states (TX, FL, NY, CA, GA, CT, MO, KS, VA, UT) require separate state DNC scrubs. Verify B2B exemption applicability per state. Source: PossibleNOW state DNC analysis.

5. **TX HB 700 registration:** Signed June 20, 2025, effective September 1, 2025. Registration with OCCC required by December 31, 2026 for entities engaged in business as of September 1, 2025. Verify whether "engaging in business" as a broker before registration triggers the prohibition provision or only the registration deadline. Source: Holland and Knight (June 2025), Secured Finance Network (June 2025), FunderIntel.

6. **CT registration:** CT Bill 1032 required registration with the Banking Commissioner by October 1, 2024. The brokerage must confirm whether it registered. If not, CT outreach is suspended until registration is obtained and back-compliance assessed. Source: Connecticut DOB portal.

7. **CA SB 362 APR re-disclosure and calculator:** Effective January 1, 2026. Prohibits using "rate" or "interest" deceptively; requires APR re-disclosure whenever a charge, pricing metric, or financing amount is stated during an application process for commercial financing of $500,000 or less. The qualification page calculator states a factor rate and total payback. Verify whether this constitutes a "statement of a charge or pricing metric" that triggers re-disclosure. Source: Buchalter (JDSupra, 2026), DFPI.

8. **NY CFDL and calculator as "offer":** NY CFDL (Article 8) applies to brokers and requires standardized disclosures when a financing offer is extended, up to $2.5M. Verify whether a calculator output presenting a total payback figure constitutes an "offer" triggering disclosure before underwriting. Source: NY DFS press release (February 2023), Chapman and Cutler.

9. **MO, FL, GA, KS broker registration or disclosure obligations:** Missouri SB 1359 (effective February 28, 2025) imposes disclosure requirements and broker registration. FL, GA, KS have disclosure regimes. Verify registration requirements versus disclosure-only requirements per state and whether a pure broker (not a provider/funder) is subject to registration or only disclosure delivery. Source: Alston Consumerfinance state survey, Onyx IQ state map.

10. **EBR exemption for prior funded book:** The 18-month established business relationship exemption for DNC-listed numbers: verify whether an MCA funding constitutes an EBR for TSR purposes, whether it covers the merchant's cell number (not just the business number), and whether state-level DNC EBR periods differ. Source: FTC TSR Q&A.

11. **TCPA opt-out revocation window:** As of April 11, 2025, consumers may revoke consent by any reasonable means; the sender has 10 business days to honor it. Verify this applies to B2B contexts and what "reasonable means" requires in terms of system logging. Source: BCLP (April 2025).

12. **CA DFPI broker licensing under CFL for MCA brokers:** SB 1235 primarily targets providers, not brokers, but SB 362's new APR-re-disclosure and deceptive-rate prohibitions apply to "providers communicating with applicants." Verify whether a pure brokerage facilitating an MCA but not extending credit is a "provider" under California Financing Law and therefore subject to DFPI licensing. Source: DFPI commercial financing disclosures page, Cloudsquare SB 362 analysis.

---

*Prepared by: Legal Compliance Checker | Pass 3 | Date: 2026-06-03 | Status: Pre-launch gate; do not initiate outreach until all checklist items are cleared and counsel has signed off on items 1-12 above.*
