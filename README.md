# Robustrade wallet transfer — implementation increment 1

A Go/PostgreSQL candidate solution scaffold. **This increment is not yet a working transfer API.** The purpose is to settle the contract, lay out the project, add database migrations, and establish initial tests before implementing the transfer service.

## Contents

```text
cmd/walletd/             # HTTP entry point (next increment)
internal/domain/         # Structural transfer-request contract and tests
internal/handler/        # HTTP transport layer (next increment)
internal/service/        # Transfer business workflow (next increment)
internal/repository/     # PostgreSQL transaction handling (next increment)
migrations/              # Initial up/down SQL and optional PostgreSQL tests
docs/DESIGN.md           # Updated design and decisions
docs/TEST_PLAN.md        # Current and pending behavioral tests
```

## Decisions

- 3 tables (`wallets`, `transfers`, `ledger_entries`). The transfer's unique idempotency key also stores the request identity.
- **Resolve the idempotency key before wallet-dependent validation**: a duplicate key with a changed nonexistent wallet should be a key conflict, not a wallet-not-found error. A preliminary lookup handles committed keys; the unique index arbitrates concurrent new requests.
- PostgreSQL `READ COMMITTED`, wallet locks `FOR NO KEY UPDATE` in ascending ID order, guarded debit, credit, ledger and terminal transfer status inside **one transaction** (next increment).
- `PENDING` exists inside the transaction; the committed state is `PROCESSED` or terminal business `FAILED`.
- Amounts and balances use **integer minor units** (`BIGINT`); no floating point, overdrafts, or self-transfers.
- Initial money is supplied as immutable seeded `opening_balance`. Subsequent test top-ups must use **a normal wallet transfer from a seeded reserve wallet**, not direct balance updates.
- Posted ledger rows are immutable, opening balances are immutable, and stored balances must reconcile to opening plus net ledger movement.
- Early structurally invalid requests and missing-wallet attempts using a **new** key are not persistently replayed; see design note.

## Running unit tests

Requires Go 1.23+ (no external dependencies are needed for this increment).

```bash
go test ./...
```

## Applying the PostgreSQL migration

Use a fresh development database, PostgreSQL and the `psql` CLI. Migrations are plain SQL and do not need a migration framework at this stage.

Linux/macOS:

```bash
export DATABASE_URL='postgres://user:password@localhost:5432/wallets?sslmode=disable'
psql "$DATABASE_URL" -X -v ON_ERROR_STOP=1 -f migrations/000001_init.up.sql
```

PowerShell:

```powershell
$env:DATABASE_URL = 'postgres://user:password@localhost:5432/wallets?sslmode=disable'
psql $env:DATABASE_URL -X -v ON_ERROR_STOP=1 -f migrations/000001_init.up.sql
```

The migration is versioned, transactional, and intended to run once on a fresh database. Do not rerun it on a populated database. The `down` file is a **destructive development reset**.

## Running PostgreSQL migration tests

The tests create a **randomized schema** and drop it afterward. Use a disposable database whose user has `CREATE SCHEMA` privileges. `psql` must be on `PATH`.

PowerShell:

```powershell
$env:WALLET_TEST_DATABASE_URL = 'postgres://user:password@localhost:5432/wallets_test?sslmode=disable'
go test ./... -v
```

Without `WALLET_TEST_DATABASE_URL`, PostgreSQL integration tests report `SKIP` rather than silently claiming the database was tested.

## Implementation status

- Contract and design corrections: complete.
- Migration, basic domain validation tests and optional SQL integration tests: included.
- HTTP server, transactional transfer implementation, key-race handling, concurrency tests: **not yet implemented**.

See `docs/TEST_PLAN.md` for the next increment. The company requests an AI usage disclosure and session transcript in the eventual PR; keep this discussion or a complete transcript available for that submission.
