---
pr: openshift/origin#31450
title: "OCPBUGS-100183: Always tear down upgrade tests after setup failures"
head_sha: 5460879a7d42c1c85e2e8f873228a7836d1c6cfc
base: main
reviewed_at: 2026-07-29T23:10:13Z
verdict: approve
---

## What this PR does
- Extracts `runUpgradeTest` in `test/extended/util/disruption/disruption.go`, registering `Teardown` via recover/defer before `Setup` runs, instead of deferring it only after `Setup` returns successfully.
- Uses a recover/re-panic pattern so a panic during `Teardown` never masks the original setup/test panic.
- Adds a guard in `GatewayAPIUpgradeTest.Teardown` (`test/extended/router/gatewayapi_upgrade.go`) for the case where `Setup` panics before `t.oc`/`t.gatewayName` are initialized — otherwise `Teardown` would nil-pointer-panic now that it can run in that scenario.
- Adds `disruption_test.go` with table-driven coverage of setup/test/teardown panic combinations.

## Findings

### [nit] Discarded teardown panic is not logged
- where: `test/extended/util/disruption/disruption.go:189-193`
- concern: When a primary (setup/test) panic already occurred, a subsequent panic from `Teardown` is silently discarded via `_ = recover()`. Intentional to avoid masking the real failure, but a genuine teardown-time resource leak in that path goes unreported. Consider `e2e.Logf` (or similar) before discarding, for diagnosability only.
- excerpt: |
    // A teardown panic must not replace the setup or test failure.
    defer func() {
        _ = recover()
        panic(primaryPanic)
    }()
    test.Teardown(ctx, f)

## Checked
- Build (`go build`), `go vet`, and `go test ./test/extended/util/disruption/...` all pass against this diff.
- `gofmt -l` clean on all three changed/added files.
- Traced all 6 panic-combination cases in `disruption_test.go` by hand against `runUpgradeTest` — call order and surfaced panic value match expectations in each case.
- `ready()` semantics unchanged: still guarded by `sync.Once`, still invoked via `chaosMonkeyAdapter.Test`'s own `defer ready()` regardless of whether `runUpgradeTest` panics before reaching its internal `ready()` call — `sem.Ready()` still signals exactly once on every path.
- `GatewayAPIUpgradeTest.Teardown` guard is scoped correctly: `t.oc` and `t.gatewayName` are set contiguously early in `Setup`, and the only panic point before that is the `o.Expect(t.precheckErr)` check at the top of `Setup`. All other delete calls in `Teardown` already tolerate missing resources via `IsNotFound`/`--ignore-not-found`, so the new guard covers exactly the previously-unhandled gap.
- Confirmed this is a necessary companion change, not scope creep: without the guard, the `disruption.go` fix would introduce a nil-pointer panic in `GatewayAPIUpgradeTest.Teardown` for the exact failure mode (setup panics before resource init) it's meant to fix.

## Open questions
- Would a debug-level log line before discarding a teardown panic (when a primary panic already occurred) be worth adding, so a teardown-side resource leak isn't completely silent in CI artifacts?
