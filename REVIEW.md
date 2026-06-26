---
pr: openshift/origin#31343
title: "NO-ISSUE: monitor/haproxy: treat all-apiservers-down test as flake"
head_sha: dd88e867eb640eddb10cf9ccb14f0c82161857bf
base: main
reviewed_at: 2026-06-26T10:30:17Z
verdict: approve
---

## Summary

`evaluateFullAPIOutages` now returns `{failure, success}` on outage detection instead of `{failure}` alone. CI classifies a test with both results as a flake rather than a hard failure. Success path (no outage) unchanged: still returns single passing JUnit. Tests updated to assert two-result structure for failure cases. Follows existing pattern from sibling `evaluateKubeApiServerAvailability` at `monitortest.go:280`.

## Findings

None.

## Checked

- Go composite literal `{Name: testName}` inside `[]*junitapi.JUnitTestCase{failure, {Name: testName}}` is valid syntax (spec: pointer elements in composite literals may elide `&T{}`); zero-value `FailureOutput` correctly marks it as pass.
- Single call site at `monitortest.go:283` uses `append(ret, evaluateFullAPIOutages(...)...)` — agnostic to returned slice length.
- Test restructuring is correct: old top-level `require.Len(t, junits, 1)` removed; per-branch length assertions added; `SystemOut` checks still target `junits[0]` (the failure).
- Pattern matches sibling function at `monitortest.go:245,280`; only difference is inlining the success struct literal vs. a named variable.
- Early-return success path at `monitortest.go:429` untouched.

## Open questions

None.
