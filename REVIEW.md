---
pr: openshift/origin#31291
title: "OPNET-603: Fail haproxy test when all kubeapi servers are down simultaneously"
head_sha: 3006971db32f070313e45d2d9a86315ee8300509
base: main
reviewed_at: 2026-06-16T15:49:09Z
verdict: approve
---

## Summary

Adds a new non-flaky JUnit test that fails when a single haproxy instance detects all kube-apiserver backends down simultaneously after the install phase. Uses a sweep-line algorithm over existing `OnPremHaproxyDetectsDown` constructed intervals. Threshold is 3 (HA control plane size). First outage per haproxy instance tolerated as install-time state. Existing individual-backend flaky test preserved unchanged.

## Findings

No blocking or should-fix findings.

### [nit] pre-existing `___` key packing
- where: `pkg/monitortests/network/onpremhaproxy/monitortest.go:261`
- concern: The `___` separator for packing two locator keys into one string is fragile (relies on the separator never appearing in node/backend names). This is pre-existing from `ConstructComputedIntervals` (line 113) and not introduced by this PR, so not actionable here. dgoodwin raised this in review; mkowalski explained the locator type limitation.
- excerpt: |
    pairKey := interval.Locator.Keys[monitorapi.LocatorOnPremKubeapiUnreachableFromHaproxyKey]
    parts := strings.SplitN(pairKey, "___", 2)

### [question] install-tolerance assumption
- where: `pkg/monitortests/network/onpremhaproxy/monitortest.go:341-343`
- concern: The `[1:]` skip assumes the first outage window is always the install phase. If haproxy restarts mid-run and the restart causes a brief full outage that happens to be the first window for a newly-appearing node, it would be silently tolerated. This seems acceptable given the monitoring setup but worth confirming with the author.
- excerpt: |
    for _, window := range outagesPerNode[node][1:] {

## Checked

- Sweep algorithm correctness: tie-breaking sorts recovery before down at equal timestamps, preventing artificial overlap
- Merge step: `!window.from.After(merged[len(merged)-1].to)` correctly merges touching and overlapping windows
- `[1:]` slice safety: `findFullAPIOutageWindows` only adds nodes with `len(merged) > 0`, so the slice is never empty
- New test is intentionally NOT marked flaky (returns `[failure]` only, not `[failure, success]`)
- Existing flaky test behavior preserved exactly
- Unit test coverage: no intervals, single/two/three backends, cross-node isolation, two separate outages, same-second boundary, backend flapping (merge), per-instance tracking, install tolerance, post-install failure, partial post-install outage
- TRT bot confirms 22/22 passes in CI
- dgoodwin's review feedback addressed: commented-out lines removed, test name updated to "should not encounter", failure message improved

## Open questions

- Is there a scenario where a haproxy node appears in the monitoring data without the install-phase outage being its first window? (e.g., haproxy pod restart, node replacement mid-test)
