---
pr: openshift/origin#31579
title: "OCPBUGS-115153: Make extension test-binary extraction architecture-aware"
head_sha: 52b1048368017eea6fdf4b5794b10522e0f5ca6d
base: main
reviewed_at: 2026-09-01T11:10:24Z
verdict: approve
---

# Review

## What this PR does

- Adds an optional architecture allowlist to payload extension-binary registry entries.
- Filters payload extension binaries before extraction, preserving entries without an allowlist.
- Uses `OCP_ARCH` when CI supplies the control-plane architecture and otherwise falls back to `runtime.GOARCH`.
- Marks the vSphere CSI driver extension as amd64-only and covers selection behavior with unit tests.

## Findings

None.

## Checked

- `OCP_ARCH` is the control-plane architecture used by OpenShift CI and is set to values such as `amd64` and `arm64`; using it fixes the cross-compiled test-binary case raised in the existing review thread.
- The architecture filter runs after the existing tag filter and before payload extraction; unrestricted entries retain their current behavior.
- The vSphere extension is omitted for arm64 and retained for amd64; no API, configuration, or externally consumed output contract changes were introduced.
- Existing review feedback from @neisw is addressed by commit `52b1048368017eea6fdf4b5794b10522e0f5ca6d`.

## Open questions

None.
