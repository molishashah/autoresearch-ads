# Learnings

Generated from experiments.jsonl. Last updated: 2026-05-08 (Reset for Cosmos campaigns).

## Confirmed Winners ✅

(none yet)

## Confirmed Losers ❌

(none yet)

## Still Running 🟡 (150-click maturity gate applies)

(none yet)

## What Converts — Headline Patterns

(no data yet — will populate after first experiments)

## What Does NOT Convert — Headline Patterns

(no data yet)

## Ad Group Tiers

(will be populated after first data pull)

## Rules (requires ≥3 data points)

(no rules yet — will emerge from experiment data)

## Deployment Notes

- RSAs are **immutable** via AdService UPDATE. Always create-new + pause-old.
- DKI headlines exceed 30-char schema limit. Can't reproduce — preserve existing DKI RSAs.

## Account CR Trend

| Cycle | Date | CR | Note |
|---|---|---|---|

## Meta-Learnings (carried forward from Intent campaigns)

1. **Negative keywords are the highest-ROI lever.** ~$355/mo saved in Intent campaigns, zero risk.
2. **Be conservative with high-performers.** Swapping a converting ad without A/B test cost ~$640.
3. **3 failed ads = audience/LP problem.** Stop iterating copy; investigate audience or landing page.
4. **CTR ≠ CR.** High CTR can mask zero conversions. Score on conversions, not clicks.
5. **~$1,836/mo addressable waste** was found in ad groups that never converted (ralph, gastown_ai, coding_agent). Monitor new groups early for this pattern.
6. **Account CR is volatile with small samples.** The 30-day window can shift dramatically. Don't overreact to single-cycle CR changes.
7. **Dual gate (150 clicks + 14 days) prevents premature scoring.** Avoids the exact mistake that caused the C2→C5 rollback cascade in Intent campaigns.
8. **Snapshot pipeline must be load-bearing.** Reading from snapshot.json (not raw MCP context) ensures reproducibility and archivability.
9. **API status enums (CRITICAL — caused a wrong report in C14): status 2 = ENABLED, status 3 = REMOVED.** Verified via `campaign.status = ENABLED` -> rows return 2, and `conversion_action.status = REMOVED` -> rows return 3. Do NOT assume 3 means ENABLED. Always filter by the named enum (e.g. `status = ENABLED`) rather than eyeballing the integer.
10. **Conversion-tracking check is the FIRST gate every cycle.** A conversion action only counts toward the `conversions` metric if it is BOTH `status = ENABLED` (2) AND `include_in_conversions_metric = true`. As of C9-C15 every include_in_conversions=true action is REMOVED (3), so live conversions cannot register. Confirm at least one ENABLED+included action exists before scoring or running any copy experiment.
