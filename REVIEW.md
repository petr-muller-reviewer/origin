---
pr: openshift/origin#31196
title: "OCPBUGS-86225: fix PDB AlwaysAllow test failure on IPv6-primary dualstack clusters"
head_sha: 9596377d687c2083dcc1bd4f895dbd2f983f8fb1
base: main
reviewed_at: 2026-05-26T16:18:26Z
verdict: approve
---

## Findings

### [nit] Inconsistent ipv6only usage compared to nearby nginx config
- where: `test/extended/testdata/poddisruptionbudgets/nginx-with-delayed-ready-deployment.yaml:70`
- concern: The sibling nginx config for port 8443 (bindata.go:51715-51716) uses `listen [::]:8443 ipv6only=on;` paired with `listen 8443;`. This PR omits `ipv6only=on` for port 8080. Without it, behavior depends on `net.ipv6.bindv6only` kernel setting. On Linux default (0), the IPv6 socket is dual-stack and could conflict with `listen 8080;`. Nginx handles this gracefully in practice, and the pattern works correctly, but the inconsistency is worth noting.
- excerpt: |
    listen 8080;
    listen [::]:8080;

## Checked
- Root cause is accurate: `listen 8080;` binds to `0.0.0.0:8080` (IPv4 only); kubelet probes via pod IPv6 address on IPv6-primary dualstack clusters; connection refused prevents pods becoming Ready
- Both the source YAML and the generated `bindata.go` are updated in sync — no regeneration gap
- The dual-directive pattern (`listen 8080; listen [::]:8080;`) is the standard nginx dual-stack approach and is correct
- Change is test-fixture only; zero production code affected
- IPv4-only clusters are unaffected: IPv4 probes still hit `listen 8080;`
- No new tests needed: the fix unblocks the existing test on IPv6-primary clusters; adding a topology-specific test is out of scope for a flake fix

## Open questions
- Should `ipv6only=on` be added for consistency with the 8443 config? (Not blocking — the fix works either way, but aligning the style would be cleaner.)
