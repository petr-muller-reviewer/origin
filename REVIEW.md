---
pr: openshift/origin#31635
title: "OCPBUGS-122386: e2e-analysis: add --skip-readiness-checks flag for degraded cluster lanes"
head_sha: cfa47d210d4eed81642aaa6513648b9dc6674043
base: main
reviewed_at: 2026-09-29T12:07:00Z
verdict: request-changes
---

# Review

## Verdict

Request changes: bound the ClusterOperator lookup in skip mode.

The flag and environment variable produce skipped JUnit cases with the expected names, and the degraded workflow passes the variable to `gather-extra`. The new skip path still waits on a cluster API request with no deadline before it writes the report. An unresponsive API can therefore prevent the skipped results that this PR is meant to provide.

## What this PR does

- Adds `--skip-readiness-checks` and recognizes `SKIP_READINESS_CHECKS=true`.
- Emits skipped JUnit cases for readiness checks and core plus discovered ClusterOperators.
- Keeps install duration reporting in the deferred JUnit report generation.
- Adds command and package tests for skipped case names and counts.

## Findings

### [should-fix] Bound the API lookup before emitting skipped results
- where: `pkg/e2eanalysis/e2e_analysis.go:70-83`
- concern: The PR promises skipped JUnit output for intentionally degraded clusters. `Run` calls this lookup before adding any skipped cases. `e2e.LoadConfig()` supplies no timeout override, so the resulting `rest.Config.Timeout` is zero; `dynamic.NewForConfig` preserves it. With `context.Background()`, client-go gives this `List` no overall deadline. If the API accepts a connection but stops responding, the fallback at lines 183-190 is never reached and `gather-extra` can stall without the promised report. Use a bounded context and fall back to the core operator names when the lookup times out.
- excerpt: |
    return dc.Resource(schema.GroupVersionResource{
        Group:    "config.openshift.io",
        Resource: "clusteroperators",
        Version:  "v1",
    }).List(context.Background(), metav1.ListOptions{})

### [nit] State the actual reason for skipping
- where: `pkg/e2eanalysis/e2e_analysis.go:171-173`
- concern: Spec: the PR calls for a clear skip message. When only `--skip-readiness-checks` is set, the JUnit message says `SKIP_READINESS_CHECKS is set`, which is false. A source-neutral message would describe both supported paths accurately.
- excerpt: |
    if opt.SkipReadinessChecks || os.Getenv("SKIP_READINESS_CHECKS") == "true" {
        skipMsg := "Skipped: cluster is running in degraded mode (SKIP_READINESS_CHECKS is set)"

### [nit] Share the ClusterOperator lookup logic
- where: `pkg/e2eanalysis/e2e_analysis.go:70-83`
- concern: Standards: possible Duplicated Code (a judgment call). The new helper repeats configuration loading, dynamic client creation, and a ClusterOperator list call already in the normal `Run` path at lines 209-220 and 258-263. Sharing that logic would keep the two paths aligned while retaining a test seam.
- excerpt: |
    cfg, err := e2e.LoadConfig()
    if err != nil {
        return nil, err
    }
    dc, err := dynamic.NewForConfig(cfg)
    if err != nil {
        return nil, err
    }

## Resolved

None.

## Checked

- The degraded workflow sets `SKIP_READINESS_CHECKS: "true"`, and `gather-extra` declares that environment variable in `openshift/release`.
- The skipped case names match the normal readiness and operator check identities, including runtime operators when listing succeeds.
- `GenerateReport` remains deferred, and it still records install durations when a JUnit directory is provided.
- `go test ./pkg/e2eanalysis ./pkg/cmd/openshift-tests/e2e-analysis` passed.
- Verified the timeout path in vendored client-go: `LoadConfig()` passes no timeout override, `dynamic.NewForConfig()` preserves `rest.Config.Timeout`, and a zero value means no overall request timeout.
- No documented repository coding standard violation was found in the changed hunks.

## Open questions

- What maximum wait for the ClusterOperator list is acceptable in `gather-extra` before using the core operator fallback?
