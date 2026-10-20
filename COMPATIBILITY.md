# Suban RPC compatibility notes

Status: canonical compatibility record (Phase 1, October 2026).

This note resolves the `getLedgerEntries` contradiction between the project plan
and the RPC README. It distinguishes the **runtime method** from the **Rust
preflight test fixture**.

## 1. `getLedgerEntries` is supported at runtime

`getLedgerEntries` is a Go JSON-RPC method. It is served by
`methods.NewGetLedgerEntriesHandler` using:

- the fast core client,
- the ledger reader.

It does **not** call the Rust preflight host. There is no runtime dependency on
`soroban-env-host` for this method. The README's support list is therefore
correct for the server.

## 2. The failing tests are preflight tests, not `getLedgerEntries`

`simulateTransaction` uses `params.PreflightGetter`, which is backed by the
vendored Rust preflight library (`soroban-env-host 23`). The test fixture
`cmd/stellar-rpc/internal/preflight/testnet-config.xdr` is a protocol-25
configuration. Host 23 cannot execute it, so `TestGetPreflight` and
`TestGetPreflightDebug` fail with:

```text
HostError: Error(Context, InternalError)
```

This is a **host/fixture protocol mismatch**. It does not mean the Go
`getLedgerEntries` handler is broken.

## 3. CI policy

- The `internal/preflight` package is still compiled (`go vet ./...`).
- Its tests are excluded from the CI gate until the host is upgraded.
- Every other package's tests, including the JSON-RPC routing and ledger-reader
  paths, remain in the gate.
- The exclusion is narrow and named; it is not a blanket `go test` skip.

## 4. Resolution path

Upgrade the Rust preflight host to a protocol-25-compatible `soroban-env-host`
release, regenerate or revalidate the fixture, and re-enable the tests.

**Acceptance criteria:**

1. `go test ./cmd/stellar-rpc/internal/preflight/...` passes locally and in CI.
2. The CI workflow no longer excludes `internal/preflight` from the test gate.
3. The `getLedgerEntries` runtime path remains covered by its handler tests.
4. `README.md` and this note agree; the plan's "known limitation" is reworded as
   a preflight test-coverage limitation, not a `getLedgerEntries` outage.

Until then, operators can use `getLedgerEntries` against a healthy Pi node, but
should treat preflight simulation coverage as reduced for protocol-25 fixtures.

## 5. References

- `cmd/stellar-rpc/internal/jsonrpc.go` — method registration.
- `cmd/stellar-rpc/internal/preflight/preflight_test.go` — failing tests.
- `README.md` — supported-method list and pointer to this note.
- [`PLAN-EXECUTION.md`](https://github.com/Pi-Defi-world/Suban/blob/main/PLAN-EXECUTION.md) — Phase 1 item 1.8.
