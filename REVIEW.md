---
pr: openshift/origin#31600
title: "OCPBUGS-115163: make upgrade acknowledgement aware of CVO payload retrieval"
head_sha: 33fc58118ba1a9b0fcec7b41c3914586b8f3d3a5
base: main
reviewed_at: 2026-09-07T12:23:40Z
verdict: approve
refresh_log:
  - at: 2026-09-07T12:23:40Z
    old_sha: 2d87a1ff82a0b993691719376bd00799497624a9
    new_sha: 33fc58118ba1a9b0fcec7b41c3914586b8f3d3a5
    summary: "Reviewed 35 additions and 10 deletions: context-aware polling, desired-update validation, diagnostic redaction, and stale-condition test coverage."
---

## Verdict

Approve. The changed acknowledgement step remains gated on `status.observedGeneration`, fails the dedicated acknowledgement JUnit case immediately for a target-matched `ReleaseAccepted=False`, and fails that same case after a bounded wait when retrieval never completes.

## What this PR does

- Recognizes the CVO's target-specific `ReleaseAccepted` condition while waiting for acknowledgement.
- Extends the short platform timeout only while retrieval of the requested payload is visibly in progress.
- Fails fast and surfaces the CVO reason when retrieval or verification is rejected, without copying the raw condition message into the JUnit failure.
- Caps a stuck retrieval at 15 minutes while retaining the existing `observedGeneration` success criterion.

Since previous review:

- Polling now propagates cancellation into ClusterVersion API reads, both during acknowledgement and completion.
- `versionMonitor.Check` validates that `spec.desiredUpdate` still matches at the exact observed-generation boundary.
- Tests now prove an image request cannot fall back to a matching version from a different image; the raw CVO message was removed from failure output.

## Findings

No findings.

## Checked

- `test/e2e/upgrade/upgrade.go:490-605`: `RecordJUnit` records errors under `[sig-cluster-lifecycle] Cluster version operator acknowledges upgrade`; the enclosing upgrade then also records `Cluster completes upgrade` as failed and returns.
- `test/extended/util/disruption/disruption.go:230-247,366-391`: additional results with a failure string become independent failing JUnit test cases in the disruption suite.
- CVO `pkg/cvo/status.go:setReleaseAcceptedCondition`: retrieval is `Unknown`, retrieval/verification failure is `False`, and successful payload loading is `True`.
- CVO scenario fixtures use the exact `version=%q image=%q` message form matched by `releaseAcceptedForTarget` for both in-progress and failed retrieval.
- A retrieval that stays `Unknown` cannot pass: it reaches `maxCVOUpdateAckTimeout`; a missing or stale other-target condition fails at the shorter platform timeout.
- All upgrade-suite variants (`all`, `platform`, and `none`) select the same disruptive cluster-upgrade Ginkgo test; their only difference is the set of parallel upgrade checks.
- `go test ./test/e2e/upgrade/` passed.
- `go vet ./test/e2e/upgrade/` passed.
- `git diff --check upstream/main...HEAD` passed.
- Refresh diff `2d87a1ff82..33fc58118b` is limited to 35 additions and 10 deletions across the same three upgrade files; the context and generation checks are sound and the new stale-condition case covers image-precedence behavior.
- CodeRabbit approved the updated head after its stale-condition coverage comment was addressed.
- Required CI is green at `33fc58118b`, including unit, lint, verify, AWS upgrade-rollback, vSphere, and rerun bare-metal IPv6 jobs; the earlier bare-metal failure was overridden as unrelated infrastructure disruption before its rerun passed.

## Open questions

- Non-blocking: would it be worth extracting the acknowledgement-state decision into a small helper so tests can assert the `ReleaseAccepted=False` and stuck-`Unknown` outcomes directly? The current unit test covers target matching, while the JUnit failure routing is established by inspection of the existing disruption harness.
