---
pr: openshift/origin#31252
title: "TRT-2617: Fix CPU metrics monitor node role lookup"
head_sha: f9c36b9d157822caf5563baa75d915addba5da8c
base: main
reviewed_at: 2026-06-03T13:00:13Z
verdict: approve
---

## Findings

### [nit] collectCPUMetricsFromPrometheusClient takes concrete type instead of interface
- where: `pkg/monitortests/testframework/cpumetriccollector/monitortest.go:109`
- concern: The new method accepts `*kubernetes.Clientset` rather than `kubernetes.Interface`. Using the interface would allow unit testing with a fake client. Low priority given this is a debugging-oriented functional test, but worth noting for future testability.
- excerpt: |
    func (w *cpuMetricCollector) collectCPUMetricsFromPrometheusClient(ctx context.Context, prometheusClient prometheusv1.API, kubeClient *kubernetes.Clientset, startTime time.Time) ([]monitorapi.Interval, error) {

### [nit] No direct unit test for the buildNodeInfoMap fix
- where: `pkg/monitortests/testframework/cpumetriccollector/monitortest.go:329-354`
- concern: The actual bug fix (adding `nodeInfoMap[node.Name] = info`) is exercised only via the env-var-gated functional test. A small table-driven unit test with a synthetic `NodeList` would catch regressions without requiring external infrastructure.

### [nit] Comment-only reformatting adds diff noise
- where: `pkg/monitortests/testframework/cpumetriccollector/monitortest.go:157,190`
- concern: The "Fallback to instance..." comments were reflowed but the text is semantically unchanged. Cosmetic only; no action needed.

## Checked
- `buildNodeInfoMap` now correctly maps by node name in addition to IP and IP:port — fixes the empty node-role column
- `nodeInfo` struct construction is deduplicated properly — single `info` value reused across all map entries for the same node
- `collectCPUMetricsFromPrometheus` correctly delegates to the new extracted method after setup
- Logger field name updated from `collectCPUMetricsFromPrometheus` to `collectCPUMetricsFromPrometheusClient` to match the new function
- Functional test properly gates on env vars with `t.Skip` — will not run in CI
- `InsecureSkipVerify: true` acceptable for test targeting internal CI Prometheus instances
- No changes to the Prometheus query, interval creation logic, or data export paths

## Open questions
- Would you consider adding a unit test for `buildNodeInfoMap` to cover the name-based lookup directly? Even a 10-line table test would lock down the fix independently of external infrastructure.
