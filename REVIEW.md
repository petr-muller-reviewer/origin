---
pr: openshift/origin#31472
title: "OCPBUGS-104561: test: exclude NTO debug pods from best-effort QoS invariant"
head_sha: 9b3f3268e87eb1b7bf12a1f383c2dd89f5ae9b73
base: main
reviewed_at: 2026-08-05T22:07:26Z
verdict: approve
---

## What this PR does
- Fixes a flake in the control-plane best-effort-QoS invariant test (`test/extended/operators/qos.go`) caused by transient `oc debug` pods created by NTO e2e tests in `openshift-cluster-node-tuning-operator`.
- Refactors the two pre-existing inline exclusions (MCO debug namespace, `openshift-debug-*` prefix) plus the new NTO namespace into a single `isKnownEphemeralDebugPod` helper.
- Adds `test/extended/operators/qos_test.go`, the first unit test for this pod-classification logic.

## Findings

### [nit] loss of per-namespace rationale comments at call site
- where: `test/extended/operators/qos.go:75-79`
- concern: the prior version had detailed inline comments per namespace explaining why each exclusion exists (including the OCPBUGS-99916 reference for `openshift-debug-*`). These were collapsed into one generic comment. Not wrong, just slightly less self-documenting for a future maintainer reading only the code.
- excerpt: |
    // Exclude only recognized transient oc debug pods in the namespaces
    // used by test helpers. They are best-effort QoS by design rather
    // than control-plane workloads.
    if isKnownEphemeralDebugPod(&pod) {
        continue
    }

### [question] coverage of label/annotation-based detection in the new namespace
- where: `test/extended/operators/qos_test.go:25-32`
- concern: the new NTO test case only exercises the legacy `-debug-` name-substring path of `isEphemeralDebugPod`, not the modern managed-by-label or source-resource-annotation paths, for this namespace specifically. `isEphemeralDebugPod` itself is unchanged and already covered elsewhere, so this is optional polish rather than a real gap.
- excerpt: |
    pod: v1.Pod{ObjectMeta: metav1.ObjectMeta{
        Namespace: nodeTuningOperatorNamespace,
        Name:      "worker-a-debug-abcde",
    }},
    want: true,

## Checked
- Traced `isKnownEphemeralDebugPod` against all three namespace branches (`node.DebugNamespace` = `openshift-machine-config-operator`, new `nodeTuningOperatorNamespace`, `openshift-debug-*` prefix) — behavior-preserving for the two existing cases, additive for the new one.
- Confirmed the exclusion still requires `isEphemeralDebugPod` (label/annotation/name-substring), so ordinary NTO pods (e.g. `tuned-*`) are not exempted — matches the PR body's claim.
- Verified all 6 new table-test cases against the implementation logic; expected results match.
- Test-only change (2 files under `test/extended/operators/`), no production code touched — low blast radius.

## Open questions
- Would it be worth restoring a one-line comment referencing OCPBUGS-99916/OCPBUGS-104561 at the call site for future debugging context, now that the rationale lives only in git history?
