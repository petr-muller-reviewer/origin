---
pr: openshift/origin#31600
title: "OCPBUGS-115163: make upgrade acknowledgement aware of CVO payload retrieval"
head_sha: 2d87a1ff82a0b993691719376bd00799497624a9
base: main
reviewed_at: 2026-09-03T10:46:13Z
verdict: approve
---

## Verdict

Approve. The changed acknowledgement step remains gated on `status.observedGeneration`, fails the dedicated acknowledgement JUnit case immediately for a target-matched `ReleaseAccepted=False`, and fails that same case after a bounded wait when retrieval never completes.

## What this PR does

- Recognizes the CVO's target-specific `ReleaseAccepted` condition while waiting for acknowledgement.
- Extends the short platform timeout only while retrieval of the requested payload is visibly in progress.
- Fails fast and surfaces the CVO reason/message when retrieval or verification is rejected.
- Caps a stuck retrieval at 15 minutes while retaining the existing `observedGeneration` success criterion.

## Findings

No findings.

## Checked

- `test/e2e/upgrade/upgrade.go:487-597`: `RecordJUnit` records errors under `[sig-cluster-lifecycle] Cluster version operator acknowledges upgrade`; the enclosing upgrade then also records `Cluster completes upgrade` as failed and returns.
- `test/extended/util/disruption/disruption.go:230-247,366-391`: additional results with a failure string become independent failing JUnit test cases in the disruption suite.
- CVO `pkg/cvo/status.go:setReleaseAcceptedCondition`: retrieval is `Unknown`, retrieval/verification failure is `False`, and successful payload loading is `True`.
- CVO scenario fixtures use the exact `version=%q image=%q` message form matched by `releaseAcceptedForTarget` for both in-progress and failed retrieval.
- A retrieval that stays `Unknown` cannot pass: it reaches `maxCVOUpdateAckTimeout`; a missing or stale other-target condition fails at the shorter platform timeout.
- All upgrade-suite variants (`all`, `platform`, and `none`) select the same disruptive cluster-upgrade Ginkgo test; their only difference is the set of parallel upgrade checks.
- `go test ./test/e2e/upgrade/` passed.
- `go vet ./test/e2e/upgrade/` passed.
- `git diff --check upstream/main...HEAD` passed.

## Open questions

- Non-blocking: would it be worth extracting the acknowledgement-state decision into a small helper so tests can assert the `ReleaseAccepted=False` and stuck-`Unknown` outcomes directly? The current unit test covers target matching, while the JUnit failure routing is established by inspection of the existing disruption harness.
