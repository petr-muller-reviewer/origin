---
pr: openshift/origin#31481
title: "Bug OCPBUGS-104497: Fix [sig-ci] prow job name OS version test for 4.23 rhcos9"
head_sha: b67e0bd4959188b4106952186d53df063d6595a4
base: main
reviewed_at: 2026-08-06T12:56:02Z
verdict: approve
pr_state: MERGED
refresh_log:
  - from: 4b6c84261115c29cb5a43e553f8f8ab01c208f6e
    to: b67e0bd4959188b4106952186d53df063d6595a4
    summary: >
      PR force-pushed (single commit). Merged the separate "upgrade" and
      "default" else-if branches into one unified `else` branch that uses
      `getInstallVersion()` for all non-explicit-marker, non-`-runc` jobs
      instead of parsing `Status.Desired.Version` separately for the
      default case. Resolves the "Duplicate ClusterVersion parsing" nit
      from the prior review. Net diff vs prior reviewed version: 4
      insertions, 9 deletions in test/extended/ci/job_names.go. No new PR
      comments or reviews since prior review.
  - from: b67e0bd4959188b4106952186d53df063d6595a4
    to: b67e0bd4959188b4106952186d53df063d6595a4
    summary: >
      No code change. CI/process activity only: petr-muller commented
      "/verified by /payload-job" (2026-08-05T22:46:34Z), openshift-ci-robot
      acknowledged the verify command (2026-08-05T22:46:45Z), and
      redhat-chai-bot requested re-tests via "/test e2e-gcp-csi" and
      "/test e2e-metal-ipi-ovn-ipv6" (2026-08-05T22:49:38Z). No review
      feedback, no findings affected.
  - from: b67e0bd4959188b4106952186d53df063d6595a4
    to: b67e0bd4959188b4106952186d53df063d6595a4
    summary: >
      No code change. petr-muller ran "/override-sticky ci/prow/e2e-gcp-csi
      ci/prow/e2e-metal-ipi-ovn-ipv6" (2026-08-05T23:04:12Z), acknowledged
      by openshift-ci[bot] (2026-08-05T23:04:19Z), overriding the two
      previously-requested CI contexts. No review feedback, no findings
      affected.
  - from: b67e0bd4959188b4106952186d53df063d6595a4
    to: b67e0bd4959188b4106952186d53df063d6595a4
    summary: >
      PR merged. redhat-chai-bot reported all tests passed
      (2026-08-06T00:20:26Z); openshift-ci-robot posted the Jira
      pre-merge-verification check for OCPBUGS-104497, confirming
      associated PRs merged and pre-merge verified, moving the Jira issue
      to MODIFIED (2026-08-06T00:20:20Z). No code change, no review
      feedback since prior refresh, no findings affected.
---

## Summary

Fixes `validateStandaloneNodeOS` in `test/extended/ci/job_names.go` so jobs with no
explicit `rhcos9`/`rhcos10`/`-runc` marker in the job name correctly expect `rhel-9`
on 4.x clusters instead of unconditionally defaulting to `rhel-10`. This was causing
a permafail on all 4.23 nightly jobs (RHCOS9 is the actual default for 4.23).

Since previous review: the author force-pushed a follow-up commit that removes the
separate `strings.Contains(jobName, "upgrade")` branch and merges it with the
default-case branch into a single unified `else`, using `getInstallVersion()`
(earliest completed ClusterVersion history entry) uniformly instead of parsing
`Status.Desired.Version` separately for the non-upgrade default case. This is a
simplification that also resolves the previously-flagged duplicate-parsing nit.

## Findings

### [resolved] Duplicate ClusterVersion.Status.Desired.Version parsing
- where: was `test/extended/ci/job_names.go:283-289` (prior review), now unified into `test/extended/ci/job_names.go:283-289` (current)
- concern (original): the `else` branch parsed `clusterVersion.Status.Desired.Version` independently from the later "OSImageStream not found" fallback path, which parses the same field again.
- resolution: the author's follow-up commit removed the separate `desiredVersion` parse entirely; the unified `else` branch now reuses `getInstallVersion(clusterVersion)`, the same helper already used by the old `upgrade`-specific branch. No behavior regression expected: for non-upgraded clusters, `getInstallVersion()` (earliest completed history entry) and `Status.Desired.Version` should report the same major version.
- excerpt: |
    } else {
        // A cluster installed with OCP 4.x uses rhel-9, whether upgraded to 5.x or not.
        // For upgrade jobs (4→5), the desired version is 5.x but the OS was installed
        // as rhel-9 and is preserved after upgrade.
        installVersion := getInstallVersion(clusterVersion)
        if installVersion.Major() < 5 {
            targetStream = "rhel-9"
        }
    }

### [nit] Stale comment on default assumption
- where: `test/extended/ci/job_names.go:276`
- concern: The comment `// If no explicit stream assume rhel-10` is no longer accurate for 4.x clusters after this fix — cosmetic only.
- excerpt: |
    // If no explicit stream assume rhel-10
    targetStream := "rhel-10"

## Checked
- The new unified `else` branch only fires for non-explicit-marker, non-`-runc` jobs (upgrade and non-upgrade alike) — correctly scoped, doesn't affect the `rhcos9`/`rhcos10`/`-runc` branches.
- `getInstallVersion()` requires at least one completed `UpdateHistory` entry (`e2e.Failf`s otherwise) — true for any real cluster including a freshly-installed one (install itself is a completed update), so no new failure mode introduced by dropping the `Status.Desired.Version` path.
- Error handling matches existing file conventions (`o.Expect(err).NotTo(o.HaveOccurred(), ...)`).
- No unit tests added, consistent with the rest of the file — no unit tests exist for this Ginkgo e2e package since it's cluster-dependent; PR description confirms `go vet`/`go test` compile cleanly.
- No new PR comments, inline comments, or reviews since prior `reviewed_at`.
- Net change vs. prior reviewed version is a simplification (4 insertions, 9 deletions), low risk of regression.

## Open questions
- None — fix is minimal, correctly scoped, matches the described root cause, and the follow-up commit is a net simplification over the previously-reviewed version.
