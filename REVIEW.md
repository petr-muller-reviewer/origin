---
pr: openshift/origin#31292
title: "OPNET-602: Fail haproxy monitor test when log collection fails"
head_sha: b4a2d0f549e3838b0d517d6a0e34160c7eb60579
base: main
reviewed_at: 2026-06-16T15:49:07Z
verdict: approve
---

## Summary

Adds platform detection to the on-prem haproxy monitor test so unsupported platforms produce JUnit skips instead of vacuous passes, and supported platforms that find no haproxy pods produce a hard failure instead of silently passing with no data.

- `PrepareCollection` now fetches the Infrastructure CR (with retry) and gates all subsequent phases via `NotSupportedError` when the platform lacks haproxy static pods.
- Supported platforms: BareMetal, OpenStack, vSphere with an API VIP and OpenShiftManagedDefault LB.
- `scanAllOperatorPods` returns an error when zero pods are found, converting a silent pass into a collection failure.
- New `platform.go` with `notSupportedPlatformReason()` and comprehensive unit tests in `platform_test.go`.

## Findings

### [nit] Zero-container gap in scanAllOperatorPods
- where: `pkg/monitortests/network/onpremhaproxy/monitortest.go:109-127`
- concern: The check guards against zero pods, but if pods exist and none contains a `haproxy` container (line 117 filter), `scanAllOperatorPods` returns nil. The label selector makes this unlikely in practice, but a counter for scanned containers would close the gap.
- excerpt: |
    if len(infraPods) == 0 {
        return fmt.Errorf("found no haproxy pods to scan: ...")
    }
    // ... later, containers are filtered by name == "haproxy"

### [nit] Test coverage for LB type on OpenStack and vSphere
- where: `pkg/monitortests/network/onpremhaproxy/platform_test.go:81-92`
- concern: User-managed loadbalancer test case only covers BareMetal. The code applies the same LB-type gating to OpenStack and vSphere. Adding equivalent test cases would prevent regression if someone modifies the switch arms independently.

### [nit] Switch case repetition in platform detection
- where: `pkg/monitortests/network/onpremhaproxy/platform.go:26-49`
- concern: The three platform branches are near-identical. Go's type system makes abstraction difficult here, so the repetition is reasonable. If a fourth platform gets haproxy support, it could be missed — but the test file makes the supported set explicit.

## Checked

- `NotSupportedError` integration follows the established pattern (matches `disruptionnewapiserver`, `disruptionserviceloadbalancer`).
- Framework handles `NotSupportedError` uniformly at every phase boundary, converting to JUnit skips.
- `RetryWithExponentialBackoff` used for Infrastructure API call; MicroShift handled via `IsNotFound`.
- Both deprecated `APIServerInternalIP` (singular) and current `APIServerInternalIPs` (plural) fields are checked.
- `WriteContentToStorage` returning `w.notSupportedReason` directly is correct: nil on supported platforms, `NotSupportedError` on unsupported.
- No race on `notSupportedReason` field: framework calls phases sequentially per monitor test.
- No variable shadowing issues: inner `err` in `PrepareCollection:43` is in a new scope, outer `err` from line 38 is no longer needed.

## Open questions

- Would it be worth also counting scanned haproxy containers (not just pods) and failing if zero were scanned, to guard against a hypothetical container rename?
- The pre-existing `// move to prepare?` comment on `StartCollection:71` — now that `PrepareCollection` does real work, is moving `kubernetes.NewForConfig` there a natural follow-up?
