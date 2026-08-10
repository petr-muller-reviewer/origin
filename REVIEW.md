---
pr: openshift/origin#31487
title: "NO-ISSUE: Consider centos-10 stream for OKD clusters"
head_sha: ff28c753e99bea9e644f757a2c4422dc75a17079
base: main
reviewed_at: 2026-08-10T17:29:28Z
verdict: approve
---

## Findings

### [nit] two Failf branches could be one XOR check
- where: `test/extended/ci/job_names.go:272-277`
- concern: The two `if isOKD && !... / if !isOKD && ...` blocks implement an XOR and could collapse to one `if isOKD != strings.Contains(...)`. Current form is arguably clearer given the distinct failure messages, so this is stylistic only.
- excerpt: |
    if isOKD && !strings.Contains(rawJobName, "okd-scos") {
        e2e.Failf("cluster is OKD but job name %q does not contain 'okd-scos'", jobName)
    }
    if !isOKD && strings.Contains(rawJobName, "okd-scos") {
        e2e.Failf("cluster is not OKD but job name %q contains 'okd-scos'", jobName)
    }

### [question] does OKD always target centos-10 regardless of install version?
- where: `test/extended/ci/job_names.go:280-298` (`validateStandaloneNodeOS`)
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

## Checked
- `validateStandaloneNodeOS(oc *exutil.CLI, jobName, rawJobName string)`: `rawJobName` (pre `e2e-` split) is used only for the `okd-scos`/`isOKD` check, since that substring lives in the job's variant, ahead of the `e2e-` split point. `jobName` (post-split) is used for the `rhcos10`/`rhcos9`/`-runc` switch and the later `rhcos9-10` mixed-cluster check, matching the split's original purpose of avoiding false substring matches from the org/repo/branch portion of the job name.
- `switch` precedence in `validateStandaloneNodeOS`: the `isOKD` case correctly takes precedence over the `rhcos10`/`rhcos9`/`-runc` substring checks, avoiding misclassification of an OKD job name that also happens to contain e.g. `rhcos10`.
- The `ClusterVersion` fetch lives entirely inside `validateStandaloneNodeOS`, which is only reached on the non-MicroShift/non-HyperShift path — MicroShift and HyperShift clusters never touch it.
- No unit tests expected/needed — this is itself a live-cluster e2e assertion, consistent with the rest of the file.
- No changes to shared library code or other call sites; blast radius limited to this one Ginkgo `It`.

## Open questions
- Has OKD ever used a stream other than `centos-10` (e.g. during an internal 9→10-style transition)? If not, worth a one-line comment explaining why the version-based fallback is skipped for OKD.
