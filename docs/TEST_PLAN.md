# Behavioral test plan

## Increment 1 — schema and request contract

- [x] Table-driven Go tests for the structurally valid transfer request and comparison of immutable details.
- [x] PostgreSQL migration integration test that isolates itself in a random test schema (runs only if `WALLET_TEST_DATABASE_URL` and `psql` are provided).
- [x] PostgreSQL tests for valid opening balances, ledger reconciliation, uniqueness, foreign keys, positive amounts, immutable opening balance, and append-only ledger.
- [x] DB test of `INSERT ... ON CONFLICT DO NOTHING` with an existing key but an incoming nonexistent destination.

**Verification distinction:** Go unit tests can run without PostgreSQL. PostgreSQL tests need a live database and are not confirmed merely by compilation. Tests of a manually posted pair of ledger entries verify schema constraints, not transfer service correctness.

## Increment 2 — service, handler and full PostgreSQL tests (not implemented yet)

- [ ] `POST /transfers` successful transfer: persisted `PROCESSED`, atomic balance updates, exactly two matching ledger rows.
- [ ] Insufficient funds: persisted `FAILED`, no ledger rows, no balance changes.
- [ ] Reserve-wallet top-up using a new transfer key, followed by retrying the original failed key; original failure is replayed.
- [ ] Repeat an accepted request: same persisted outcome and no duplicate side effects.
- [ ] Existing idempotency key with changed recipient that doesn't exist: conflict (`409`) before missing-wallet response (`404`).
- [ ] Concurrent same-key requests: only one transfer and two ledger entries.
- [ ] Concurrent distinct keys oversubscribing a source wallet: no overspending.
- [ ] Concurrent A→B and B→A operations: no lock-order inversion/deadlock.
- [ ] Simulated error before commit: no partial wallet/ledger/transfer updates.
- [ ] Simulated lost response after commit: subsequent retry replays committed result.
- [ ] Commit outcome unknown: client uses same key rather than creating a new logical request.
- [ ] Arithmetic overflow on receiving wallet: transaction rolls back.
- [ ] Reject JSON fractions, invalid amount types, malformed body, missing fields.
- [ ] End-of-test reconciliation across all wallets, and conservation of total funds.

## Testing funding without violating accounting

Seed wallets `A` (opening 50), `B` (opening 0), and `reserve` (opening 1000). Test `K1` (A→B, 100) and verify `FAILED`; transfer `K2` (reserve→A, 200) and verify two ledger entries; retry `K1` and verify it is still `FAILED`; transfer `K3` (A→B, 100) and verify it succeeds. All balances must equal opening balance plus posted net ledger movements.
