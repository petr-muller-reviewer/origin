---
pr: openshift/origin#31457
title: "OCPBUGS-100385: MonitorTest: measure the union of ClusterOperator wait intervals"
head_sha: 4a51d5701d41c40cea791465f1e072fbf361304d
base: main
reviewed_at: 2026-08-03T10:17:24Z
verdict: approve
---

## Summary

Replaces `fromAndTo` (latest-end minus earliest-start across all CVO-Progressing intervals naming an operator) with `summarizeWaitingIntervals`, which sorts intervals, merges overlapping/adjacent ones into episodes, and sums only episode durations. Fixes overcounting when CVO reports waiting on an operator across multiple disjoint episodes (e.g. due to CVO restarts), which previously counted the gap between episodes as "waiting" time. Also enriches failure/diagnostic output with total duration, episode count, longest episode, full span, and per-episode ranges. Scope: `pkg/monitortests/clusterversionoperator/legacycvomonitortests/operators.go` + its test file only.

## Findings

### [question] verbosity of diagnostic string in both flake and hard-failure paths
- where: `pkg/monitortests/clusterversionoperator/legacycvomonitortests/operators.go:643-647`, `:688-690`
- concern: `waitingIntervalSummary.String()` always renders total, episode count, longest, full span, and every episode's range. This is used both for the "under 3 minutes" (flake) message and the "waited over 3 minutes" (failure) message — intentional per PR description for RCA, but worth the author confirming this verbosity is desired in both paths rather than only the failure path.
- excerpt: |
    summary := summarizeWaitingIntervals(intervals)
    if summary.total < 3*time.Minute {
    	return fmt.Sprintf("%s completing its update within less than three minutes: %s", co, summary)
    }

## Checked
- Merge algorithm correctness by hand-tracing: sort by (From, To) ascending, then merge when `interval.From` is not strictly after the last episode's `To` — correctly handles overlap, adjacency, and full containment (nested interval doesn't shrink `To` since only `max` applied).
- No aliasing bug: `sorted := append(monitorapi.Intervals(nil), intervals...)` copies before sort; `last := &summary.episodes[...]` mutates only the freshly built `episodes` slice.
- Invalid interval filtering (`From.IsZero()` or `To.Before(From)`) happens inside the merge loop after sort — doesn't affect sort correctness, correctly excludes garbage.
- Zero-duration "point" intervals (`From == To`) are treated as valid zero-length episodes, matching the "separated point does not fill gap" test intent.
- `fromAndTo` fully removed; `grep -rn fromAndTo` across the repo returns nothing — no dangling references.
- `grep` for `COWaiting`/`summarizeWaitingIntervals`/`waitingIntervalSummary` outside this package returns nothing — change is fully self-contained.
- Hand-verified `Test_updateCOWaiting` updated expectations (e.g. "incremental one": old `next + 2*time.Minute` spanning a 3-hour gap → new `5 * time.Minute` = 3min + 2min, correctly excluding the gap) — matches the bug being fixed.
- `Test_summarizeWaitingIntervals` table-driven cases (disjoint point vs. real episode, overlap+adjacency with out-of-order input, multiple episodes without double-counting, invalid-interval filtering, all-empty/all-invalid) hand-traced against the implementation — all consistent.
- Could not run `go vet`/`go test` directly (no `go` binary in this sandbox) — verification is by code inspection, not an executed test run.

## Open questions
- Can CI confirm `go vet ./pkg/monitortests/clusterversionoperator/legacycvomonitortests` and `go test ./pkg/monitortests/clusterversionoperator/legacycvomonitortests` pass? I verified logic by inspection only, not execution.
- Is the full verbose `String()` output (including all episode ranges) intended for the sub-3-minute flake message too, or only the hard-failure path?
