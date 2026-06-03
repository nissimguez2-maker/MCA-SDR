# MCA SDR Plan Audit

A chained, multi-lens audit of the SDR operating-system plan. Three passes (market, strategy, implementation), a reconciliation after each, and a final plan that ties them together.

## Files

- `plan.md`: the plan under audit. Edit this and rerun whenever the plan changes.
- `AUDIT-RUNBOOK.md`: the procedure. Lists all 18 lenses with their persona file paths in the agency-agents repo, the three passes, both reconciliations, and the scope ruling. Each lens reads its persona file from that repo before auditing (Step 0 loads them, by clone or by raw URL).
- `outputs/pass1`, `outputs/pass2`, `outputs/pass3`: one audit file per lens, written during the run.
- `outputs/final/scope-ruling.md`: the scope authority's ruling.
- `market-brief.md`, `strategy-brief.md`: the narrowed briefs produced between passes.
- `FINAL-PLAN.md`: the end result.

## How to run

Open this repo in Claude Code on the web (or any Claude Code session) and paste:

```
Read AUDIT-RUNBOOK.md and plan.md in full, then execute the audit exactly as
AUDIT-RUNBOOK.md describes. Work one numbered step at a time: write the output
files for that step, show me what you produced, and wait for me to say "continue"
before moving to the next step.
```

It stops for review after the market brief (Step 2) and the strategy brief (Step 4). Read and tighten those before continuing, since that is where the chain narrows.

## The chain

```
Pass 1 Market  -> reconcile -> market-brief
Pass 2 Strategy (reads market-brief) -> reconcile -> strategy-brief
Pass 3 Implementation (reads strategy-brief) -> scope ruling -> final reconcile -> FINAL-PLAN
```

## Notes

- Keep this repo private.
- The compliance lens lists claims to verify (TCPA, DNC, state MCA disclosure) rather than asserting law. Confirm those with a live source before acting.
- Constraints baked into every lens: one operator, built off-hours, under 300 USD per month, must not become an overbuilt agent lab.
