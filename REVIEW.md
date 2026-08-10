---
pr: openshift/origin#31487
title: "NO-ISSUE: Consider centos-10 stream for OKD clusters"
head_sha: f32f8cd93f4cd628342bb47fc6d6f4f6685aafbf
base: main
reviewed_at: 2026-08-07T14:51:11Z
verdict: needs-discussion
---

## Summary

Adds OKD detection to the `[sig-ci] [Early] prow job name` / "should match os version" Ginkgo test in `test/extended/ci/job_names.go`. Detects OKD via `ClusterVersion.Status.Desired.Version` containing `okd-scos`, fails if job name and cluster OKD-ness disagree, and threads `isOKD`/`clusterVersion` into `validateStandaloneNodeOS` so OKD clusters target `centos-10` instead of falling through the RHCOS 9/10 heuristics.

## Findings

### [should-fix] ClusterVersion fetch moved ahead of isMicroShift check
- where: `test/extended/ci/job_names.go:184-194` (new code before the `isMicroShift`/`isHyperShift` branch)
- concern: The new `oc.AdminConfigClient().ConfigV1().ClusterVersions().Get(context.Background(), "version", metav1.GetOptions{})` call (with `o.Expect(err).NotTo(o.HaveOccurred())`) now runs unconditionally, before checking `isMicroShift`. Previously this fetch only happened inside `validateStandaloneNodeOS`, i.e. only on the non-MicroShift/non-HyperShift path. Existing code in this repo treats "MicroShift" and "safe to fetch ClusterVersion" as mutually exclusive — `test/extended/apiserver/api_requests.go:44-51` explicitly gates its `ClusterVersion` fetch behind `!isMicroShift`, and `IsMicroShiftCluster`'s doc comment (`test/extended/util/framework.go:2315-2325`) explains MicroShift is detected via a `microshift-version` configmap rather than `ClusterVersion`, implying MicroShift doesn't reliably serve that resource. If MicroShift clusters don't serve the `ClusterVersion` "version" singleton, this test will now hard-fail on MicroShift instead of reaching `validateMicroshiftNodeOS`'s skip/short-circuit path as before.
- excerpt: |
    clusterVersion, err := oc.AdminConfigClient().ConfigV1().ClusterVersions().Get(context.Background(), "version", metav1.GetOptions{})
    o.Expect(err).NotTo(o.HaveOccurred(), "Error getting ClusterVersion version singleton")

    isOKD := strings.Contains(clusterVersion.Status.Desired.Version, "okd-scos")
    if isOKD && !strings.Contains(originalJobName, "okd-scos") {
        e2e.Failf("cluster is OKD but job name %q does not contain 'okd-scos'", originalJobName)
    }
    if !isOKD && strings.Contains(originalJobName, "okd-scos") {
        e2e.Failf("cluster is not OKD but job name %q contains 'okd-scos'", originalJobName)
    }

    isMicroShift, err := exutil.IsMicroShiftCluster(oc.AdminKubeClient())

### [question] does OKD always target centos-10 regardless of install version?
- where: `test/extended/ci/job_names.go:277-291` (`validateStandaloneNodeOS`)
- concern: For `isOKD == true`, the function hardcodes `targetStream = "centos-10"` unconditionally. Non-OKD clusters get a version-based fallback (4.x installs preserved as rhel-9 even after upgrade to 5.x) via the `default:` branch, but OKD skips that fallback entirely. If OKD ever had a pre-centos-10 stream (mirroring the RHCOS 9→10 transition), an OKD cluster installed on that older stream and not yet upgraded would fail this check. Not necessarily wrong, but not explained in the PR description or in a comment.
- excerpt: |
    switch {
    case isOKD:
        targetStream = "centos-10"
    case strings.Contains(jobName, "rhcos10"):
        targetStream = "rhel-10"
    case strings.Contains(jobName, "rhcos9"):
        targetStream = "rhel-9"
    case strings.Contains(jobName, "-runc"):
        targetStream = "rhel-9"
    default:

### [nit] two Failf branches could be one XOR check
- where: `test/extended/ci/job_names.go:187-192`
- concern: The two `if isOKD && !... / if !isOKD && ...` blocks implement an XOR and could collapse to one `if isOKD != strings.Contains(...)`. Current form is arguably clearer for distinct failure messages, so this is stylistic only, not blocking.
- excerpt: |
    if isOKD && !strings.Contains(originalJobName, "okd-scos") {
        e2e.Failf("cluster is OKD but job name %q does not contain 'okd-scos'", originalJobName)
    }
    if !isOKD && strings.Contains(originalJobName, "okd-scos") {
        e2e.Failf("cluster is not OKD but job name %q contains 'okd-scos'", originalJobName)
    }

## Checked
- `originalJobName` (pre `e2e-` split) vs `jobName` (post-split) usage: using `originalJobName` for the `okd-scos` match is correct since `okd-scos` appears in the job-name prefix, before the `e2e-` marker.
- `switch` precedence in `validateStandaloneNodeOS`: `isOKD` case correctly takes precedence over `rhcos10`/`rhcos9`/`-runc` substring checks, avoiding misclassification of an OKD job name that also happens to contain e.g. `rhcos10`.
- No unit tests expected/needed — this is itself a live-cluster e2e assertion, consistent with the rest of the file.
- No changes to shared library code or other call sites; blast radius limited to this one Ginkgo `It`.

## Open questions
- Does MicroShift reliably serve the `config.openshift.io/v1 ClusterVersion` "version" singleton? If not, the reordered fetch will break MicroShift runs of "should match os version" — please confirm or move the fetch back inside the standalone branch.
- Has OKD ever used a stream other than `centos-10` (e.g. during an internal 9→10-style transition)? If not, worth a one-line comment explaining why the version-based fallback is skipped for OKD.
