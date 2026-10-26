# Wallet Transfer Service — Design Note

**Status:** Updated contract for implementation increment 1 (scaffold, migrations, initial tests).  
**Scope:** Robustrade's `POST /transfers` assignment.  
**Guiding principle:** Prefer a small, synchronous, auditable implementation with strong database-backed correctness over additional infrastructure.

## 1. Scope and behavioral contract

A transfer moves a positive integer amount from one existing wallet to another in the same currency. Successful operations atomically update both wallet balances, create exactly two matching ledger entries, and record a `PROCESSED` transfer. A transfer declined for insufficient funds records a terminal `FAILED` transfer, makes no balance changes, and posts no ledger entries.

The API requires a nonempty `idempotencyKey` as a **proposed API assumption**. For an accepted transfer attempt, reusing the same key with the same request details returns the same persisted result with no new financial effects. Reusing the key with different details returns an idempotency conflict without changing funds. Only structurally invalid requests are rejected before key resolution. For a structurally valid request, an already-committed idempotency key takes precedence over the existence of wallets in the incoming payload. A previously claimed key with a different destination (even a nonexistent one) therefore returns a key conflict. A fresh key referring to nonexistent wallets is rejected without a committed transfer; this rejection is outside durable replay.

The payload's `amount` is a **positive integer of the currency's smallest unit** (e.g., cents or paise, depending on the chosen currency); floating-point monetary values are not used. The assignment does not specify a currency or amount unit; these must be documented in the API README.

Suggested HTTP response mapping (not mandated by Robustrade):

| Result | HTTP | Persisted? | Replay behavior |
|---|---|---|---|
| Successful transfer | `201` | `PROCESSED` with ledger | Replay same HTTP status and persisted response |
| Insufficient funds | `409` | `FAILED`, failure reason, no ledger | Replay the same failure, even if balance later increases |
| Same key, different payload | `409` | No additional transfer | Reject without side effects |
| Invalid input | `400` | No transfer | Not covered by durable replay |
| Wallet not found for a genuinely new key | `404` | No transfer | Not covered by durable replay |
| Existing key reused with a nonexistent destination | `409` | Original transfer unchanged | Key conflict takes priority over wallet existence |
| Database/server outage | `5xx` or timeout | May be uncommitted or outcome unknown | Retry with **same key**; determine committed outcome before doing any new business effect |

The response includes only immutable or persisted transfer information (`transferId`, status, amount, wallet IDs, optional `failureReason`, and optionally `createdAt`), **not live wallet balances**. This allows deterministic replay. If exact byte-identical replay becomes mandatory, persist the original HTTP status/body, potentially warranting a separate idempotency table.

## 2. Proposed schema (conceptual; no SQL yet)

### `wallets`

| Column | Type | Rules |
|---|---|---|
| `id` | `TEXT` | Primary key; values such as `wallet_1` |
| `opening_balance` | `BIGINT` | Not null; nonnegative; supplied by controlled seed data; immutable thereafter |
| `balance` | `BIGINT` | Not null; nonnegative; initially equals opening balance |

No wallet creation, funding, deletion, overdraft, or multi-currency API is in scope. `opening_balance` is a documented baseline for test funds, not a ledger-posted funding event. The schema uses a small trigger to reject updates to `opening_balance`. Seed a reserve wallet with its own opening balance to fund other test wallets by normal wallet-to-wallet transfers.

### `transfers`

| Column | Type | Rules |
|---|---|---|
| `id` | `UUID` | Primary key |
| `idempotency_key` | `TEXT` | Not null, nonempty, unique across this API |
| `from_wallet_id` | `TEXT` | Not null; foreign key to `wallets(id)` |
| `to_wallet_id` | `TEXT` | Not null; foreign key to `wallets(id)`; different from source |
| `amount` | `BIGINT` | Not null; greater than zero |
| `status` | `TEXT` | Not null; one of `PENDING`, `PROCESSED`, `FAILED` |
| `failure_reason` | `TEXT` | Required for `FAILED`; null otherwise |
| `created_at` | `TIMESTAMPTZ` | Not null; set on first insert |

Recommended constraints: unique(`idempotency_key`), check(`from_wallet_id <> to_wallet_id`), positive amount, allowed status, and consistent status/failure-reason combination. The unique key creates the necessary lookup index. Wallet foreign keys use a restrictive/no-cascade deletion policy; the service itself does not delete wallets. Conflicting keys must be resolved before wallet existence checks, including for an incoming nonexistent wallet.

### `ledger_entries`

| Column | Type | Rules |
|---|---|---|
| `id` | `BIGINT` identity | Primary key |
| `transfer_id` | `UUID` | Not null; foreign key to `transfers(id)` |
| `wallet_id` | `TEXT` | Not null; foreign key to `wallets(id)` |
| `entry_type` | `TEXT` | Not null; `DEBIT` or `CREDIT` |
| `amount` | `BIGINT` | Not null; greater than zero |
| `created_at` | `TIMESTAMPTZ` | Not null |

Add **unique(`transfer_id`, `entry_type`)** to permit at most one `DEBIT` and one `CREDIT` per transfer; index `wallet_id` to support balance reconciliation. A small database trigger rejects ledger updates/deletes, including manual accidental mutation. No runtime funding operation is added.

**Limit of declarative constraints:** Primary keys, foreign keys, unique constraints, and row-level checks cannot by themselves enforce that a `PROCESSED` transfer has **at least** one debit and one credit, that their wallet IDs match the transfer, that both amounts equal the transfer amount, or that all wallet balances reconcile to ledger history. Those are cross-row/cross-table rules to enforce in the atomic write path and verify in real-database tests. A deferred constraint trigger could harden these rules later, but is not planned for this short assignment.

## 3. Core invariants

1. For each `PROCESSED` transfer: exactly one source `DEBIT` and one destination `CREDIT`, both of amount `transfers.amount`.
2. For each `FAILED` transfer: no financial postings and no balance changes.
3. Wallet balances never go below zero (no overdrafts).
4. For every wallet, after committed operations: `balance = opening_balance + SUM(credits) - SUM(debits)`.
5. For the transfer-only system, `SUM(wallet.balance) = SUM(wallet.opening_balance)`.
6. A given `idempotency_key` has at most one committed transfer and one terminal business outcome.
7. `PROCESSED` and `FAILED` are terminal; `PENDING` can transition only to one of them.

## 4. State-machine interpretation

The service inserts the transfer as `PENDING`, performs all processing **inside the same PostgreSQL transaction**, and changes it to `PROCESSED` or `FAILED` before commit. The final update is guarded by `status = PENDING` and must affect one row.

Other sessions normally only observe committed terminal states; `PENDING` is a real **in-transaction** state, not a durable queue/workflow state. A crash before commit removes the entire attempt, including the pending row. This is an intentional assumption because the assignment does not explicitly require asynchronous processing or externally observable pending transfers. If durable `PENDING` is required, the design needs multiple transactions, recovery, and in-progress deduplication.

## 5. Complete transaction lifecycle

**Structural validation before database access:** Decode the request and check required fields, nonempty trimmed idempotency key and wallet IDs, distinct IDs, and positive integer amount. **Do not look up wallet existence first.** A structurally invalid request returns `400` without claiming a key; accepted idempotency semantics are limited to structurally valid requests.

**Idempotency-first accepted attempt (PostgreSQL `READ COMMITTED`):**

1. A cheap **preliminary lookup** by `idempotency_key` may return an existing committed transfer. If found, compare immutable request details **before inspecting whether the incoming wallets exist**: identical details replay the stored outcome; different details return `409` key conflict. This lookup is only a fast path and is not trusted to settle concurrent requests.
2. **Begin transaction.** Attempt to insert `transfers` with `status = PENDING` and the request details using `INSERT ... ON CONFLICT (idempotency_key) DO NOTHING RETURNING id`. This must be done **before standalone wallet-existence queries**. A unique-index conflict arbitrates concurrent first-time requests. PostgreSQL foreign-key checks guard newly inserted rows; a losing `DO NOTHING` insert does not create the incoming invalid wallet reference.
3. **If the insert did nothing:** issue a new statement under `READ COMMITTED` to load the now-committed transfer, compare all immutable request fields and replay the persisted result or return `409` conflict. Never lock/update wallets on this path. The preliminary lookup and this branch jointly prevent the original bug (an unknown destination overshadowing existing key conflict). If a concurrent winner rolled back, this transaction may instead insert and become the new claimant.
4. **If this is a new transfer row:** the wallet foreign keys ensure both wallet IDs exist; map foreign-key rejection to `404` for a genuinely new key (transaction is rolled back). Acquire `FOR NO KEY UPDATE` locks on **both wallets in ascending wallet-ID order**, using an acquisition pattern that actually locks in that order. Read balances only after acquiring locks. We do not delete wallets at runtime.
5. **If sender funds are insufficient:** guarded update `PENDING -> FAILED`, persist reason `INSUFFICIENT_FUNDS`, commit, and return the persisted failure. Insert no ledger rows and modify neither balance.
6. **If funds suffice:** conditional debit requiring `balance >= amount`, then atomic credit. Assert exactly one row affected for each. Any unexpected result or arithmetic overflow aborts the transaction.
7. Insert exactly two ledger rows: source `DEBIT` and destination `CREDIT`, each equal to the transfer amount. Any error aborts the transaction.
8. Guardedly transition `PENDING -> PROCESSED` (`WHERE status = 'PENDING'` and one affected row).
9. **Commit.** Only after confirmed commit, return a deterministic response constructed from persisted transfer data.

**Key precedence guarantee:** Even if the incoming destination doesn't exist, a committed existing key with a different payload is an idempotency conflict. This applies to sequential and concurrent duplicate arrivals. A PostgreSQL integration test explicitly checks that `ON CONFLICT DO NOTHING` skips an incoming invalid wallet reference when a matching key already exists; runtime concurrency behavior will be separately tested when the service exists.

The conditional debit is defense in depth: sorted wallet locks are the primary concurrency protocol; the conditional update also enforces the balance predicate at write time. `FOR NO KEY UPDATE` is compatible with the `FOR KEY SHARE` locks generally needed by foreign-key checks on the wallet's key, unlike the stronger `FOR UPDATE` lock. Both wallet balance writers must follow the same ordered locking protocol.

For two opposite-direction transfers, both acquire the smaller wallet ID first, then the larger, regardless of which is the source. This avoids the classic A-then-B/B-then-A lock-order deadlock. It does not make all possible database deadlocks impossible.

## 6. Duplicate requests and uncertain outcomes

- **Simultaneous requests, same key:** PostgreSQL's unique index arbitrates the competing inserts. One operation can commit; the other waits for that outcome. If the first commits, the second reads and replays it; if the first rolls back, the second may claim the key and process it. At `READ COMMITTED`, load the existing row in a fresh statement after `ON CONFLICT DO NOTHING` reports no insertion.
- **Same key, different body:** compare persisted fields; reject with a distinct idempotency conflict; never silently replay a different transfer as if it matched.
- **Commit succeeded, HTTP response lost:** the committed transfer and its complete financial effects exist together. The retry discovers the existing key and reconstructs the same result, without touching balances again.
- **Known rollback (before commit):** no transfer, balance, or ledger writes survive. The key is not reserved; a retry may execute.
- **Commit outcome unknown (connection lost around `COMMIT`):** do not report an authoritative financial failure. The caller retries using the **same key**; the database's unique constraint and persisted transfer resolve whether the first operation committed. If the database is unavailable, report a retryable/unknown outcome, not `FAILED`.
- **Deadlock `40P01` or serialization failure `40001`:** roll back and, if appropriate, retry the **entire** transaction with bounded attempts and backoff. Never retry only a single SQL statement after the transaction has been aborted.
- **Database error during debit, credit, ledger or status update:** abort the entire transaction; never commit a half-finished transfer.

## 7. Constraint responsibilities vs application responsibilities

| Guarantee | Enforced by PostgreSQL constraints/transactions | Still requires service logic/tests |
|---|---|---|
| Transfer and wallet IDs are unique and references exist | Primary/foreign keys | Meaningful API errors |
| One committed transfer per idempotency key | Unique constraint | Matching request comparison, result replay, unknown-commit handling |
| Positive transfer and ledger amounts | `CHECK`, `NOT NULL` | Correct amount units, API validation |
| Source and recipient must differ | Row-level `CHECK` | User-friendly validation error |
| Nonnegative stored wallet balance | `CHECK`; conditional debit update | Acquire locks in order; interpret update count |
| Both balances, ledger and transfer outcome commit together | One ACID transaction | Place all relevant writes inside the transaction |
| No more than one entry of each type per transfer | `UNIQUE(transfer_id, entry_type)` | Ensure **both** correct entries exist for processed transfers |
| Debit/credit amounts and wallet mappings are correct | Not enforced by basic FK/CHECK/UNIQUE | Business logic and integration/reconciliation tests |
| Opening balances never change | Immutability trigger rejects changes | Ensure seeding uses INSERT and ordinary transfers never update opening_balance |
| Only allowed state transitions occur | Allowed status values via `CHECK` | Guarded updates, no terminal-state rewrites; tests |
| Original API result is replayable | Persisted terminal transfer data | Stable response mapping; no live balance fields |

## 8. Reconciliation

For each wallet compute `opening_balance + total posted credits - total posted debits` and compare with the stored current balance. The expected difference is zero. Also verify each processed transfer has exactly the expected debit/credit pair and each failed transfer has none.

Reconciliation **detects**, rather than prevents, data discrepancies. For the assignment, automated test assertions are sufficient; no background reconciliation service is planned. If reconciliation fails, do not automatically overwrite financial balances without investigating.

## 9. Test strategy (prefer real PostgreSQL for DB guarantees)

- Successful transfer: both balances correct, one debit and one credit, committed `PROCESSED`.
- Insufficient funds: committed `FAILED`, unchanged balances, zero ledger rows; seed a `reserve` wallet, make a separate successful transfer `reserve -> source` with a new key, and verify replay of the original failed key stays `FAILED` without changing any balances.
- Invalid amount, same wallet, missing wallet: rejected; no committed financial writes.
- Same key sequential retries: same transfer ID, same status/result, no additional ledger rows.
- Same key concurrent retries (many goroutines/processes): only one logical transfer; two ledger entries total.
- Same key with different payload, including a destination wallet that does not exist: key conflict takes priority over wallet validation; no financial effects.
- Two different keys simultaneously spend the same balance: never overspend; only affordable debits succeed.
- Opposite-direction transfers: no lock-order deadlock when both affordable; final balances reconcile.
- Multiple unrelated wallet pairs: no unnecessary shared wallet serialization.
- Induced SQL error or explicit rollback before commit: no persisted partial changes; retry with the same key can succeed.
- Commit followed by simulated loss of HTTP response: retried key returns committed result.
- Persisted results remain replayable after a service restart.
- Schema negative tests: duplicate key, duplicate `DEBIT`, invalid amounts, invalid foreign keys, negative balance, opening-balance mutation, and ledger mutation are rejected by PostgreSQL. Also verify that a conflict on an existing key does not fail the incoming nonexistent-destination foreign key.
- Reconciliation: stored balances equal opening balances plus ledger movements and total funds are conserved.

Service-level unit tests cover validation and outcome mapping; integration tests exercise row locks, unique-key races and transaction rollback. Mock-only concurrency tests do not prove PostgreSQL behavior.

## 10. Operational scope and open assumptions

- A single PostgreSQL database is authoritative; no queue, distributed transaction, external payment processor, or cross-service write is involved.
- Wallets are seeded once for the exercise, including a reserve wallet with a controlled initial balance. To test later funding, transfer from reserve using the ordinary API and a new key; never directly UPDATE balances. There is no runtime wallet funding API. No wallet IDs are modified or deleted during transfers.
- One currency and one consistent minor-unit convention apply to all wallets. Amounts and balances use `BIGINT`, not floating point; out-of-range values are rejected.
- No overdraft; failure for insufficient funds is terminal for that idempotency key.
- Idempotency keys are globally unique for this API and retained indefinitely for the exercise. A real multi-tenant service would need key scoping and a retention policy.
- An idempotency key is required for accepted requests; early validation errors (such as invalid syntax) are **not persisted**, and missing wallets for **new keys** are not persisted. If the reviewer requires replay of *every* response including pre-execution errors, a separate idempotency record or a more permissive transfer-attempt model would be required.
- Intermediate `PENDING` state is not externally visible; durable pending processing is out of scope.
- Database triggers reject ledger edits/deletions and opening-balance changes. A real service would additionally restrict database role permissions and audit privileged schema modifications.
- Log a transfer identifier, outcome, and retry/conflict/deadlock events for troubleshooting. Do not log secrets or unnecessary financial data. Background metrics/monitoring are optional.

## 11. Main trade-offs

**Why the design should work:** One transaction eliminates partial financial commits; a unique key handles repeated requests; ordered wallet locks handle concurrent spending; a stored balance avoids costly aggregation on every read; ledger history and reconciliation support auditability.

**Where it could fail:** Privileged/uncontrolled writers may still change current balances without ledger entries; ordinary constraints cannot fully enforce ledger-to-transfer equality; incorrect handling of unknown commit outcomes can cause false failure reporting; persisted `PENDING` and exact replay of every pre-validation HTTP error are deliberately not supported.

**Decision:** Use three tables (`wallets`, `transfers`, `ledger_entries`), synchronous in-transaction state transitions, PostgreSQL `READ COMMITTED`, sorted `FOR NO KEY UPDATE` wallet locks, a conditional debit safeguard, and durable idempotency in `transfers`. Revisit only if the assignment reviewer clarifies stronger lifecycle, replay, or accounting requirements.

**Sources:** Robustrade [`ASSIGNMENT.md`](https://github.com/Robustrade/wallet-transfer-assignment/blob/main/ASSIGNMENT.md), [`evaluation_guide.md`](https://github.com/Robustrade/wallet-transfer-assignment/blob/main/evaluation_guide.md); PostgreSQL documentation on constraints, locking, transaction isolation and `INSERT ... ON CONFLICT`.
