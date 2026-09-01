---
pr: openshift/origin#31583
title: "OCPBUGS-115163: Avoid false upgrade acknowledgement timeout during payload retrieval"
head_sha: 56c6509e62965c0051d7fbbb0a2417f646c5a161
base: main
reviewed_at: 2026-09-01T12:41:06Z
verdict: approve
---

# Review

## What this PR does

- Records the pre-request `ReleaseAccepted` condition and CVO events.
- Accepts a fresh, target-matched `RetrievePayload` signal before observed generation advances.
- Keeps event lookup optional and rejects failed or stale retrieval signals.
- Covers image-specific and version-only target matching.

## Findings

None.

## Checked

- `test/e2e/upgrade/monitor.go:74-112` verifies the desired update before accepting either the established observed-generation acknowledgement or the new signals.
- `test/e2e/upgrade/upgrade.go:489-540` takes the baseline before the patch and does not turn Event API failures into an acknowledgement failure.
- `test/e2e/upgrade/monitor.go:82-109` requires the expected CVO object, source, normal `RetrievePayload` event, exact quoted target fields, and a changed event resource version.
- `test/e2e/upgrade/monitor_test.go:14-230` covers stale, unrelated, failed, version-only, and image-specific condition/event cases.
- The sole substantive automated review comment about version-only matching is addressed by `56c6509e62`.
- `go test ./test/e2e/upgrade` and `git diff --check upstream/main...HEAD` completed successfully.

## Open questions

None.
