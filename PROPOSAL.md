# tauorm — project proposal

A SQLAlchemy-style SQL toolkit + ORM for Tauraro: a dialect-agnostic **Core**
(engine, connection pooling, a composable SQL expression language, schema
definition) plus a declarative **ORM** (mapped classes, sessions, unit of
work, relationships) built on top of it — the same two-layer split
SQLAlchemy itself uses, and for the same reason: plenty of consumers only
ever want Core (build queries, get rows back, no object mapping), and Core
has to be solid and dialect-agnostic before an ORM can be built on it
honestly.

## Why a dialect layer, not a direct dependency on one driver

This workspace already has two real, hand-verified drivers — `taupostgres`
and `tausqlite3` — and they intentionally do **not** share a common
interface today; each was designed against its own C library's natural
shape. Comparing them concretely is exactly why tauorm needs a real
abstraction boundary, not a thin wrapper over one of them:

| | taupostgres | tausqlite3 |
|---|---|---|
| Placeholder style | `$1, $2, ...` | `?` (positional, no numbering) |
| Parameter binding | one call, `Vec[str]` of all params (text) or `_binary` for fixed-width numerics | typed per-slot: `bind_int`/`bind_str`/`bind_float`/`bind_null` |
| Result shape | `exec_params()` materializes the *whole* result set up front (`PgResult` + `Row.get_*`) | step-cursor: `Statement.step()` advances one row at a time, read via `column_*` |
| Errors | `PgError { sqlstate, message }` | `SqliteError` (result-code based) |
| Async | `send_query`/`next_result` exist, but PROPOSAL.md is explicit that `next_result` still blocks on the socket like `PQgetResult` does — not real non-blocking I/O yet | none — SQLite's C API is inherently synchronous |
| Pooling | `Pool` (checkout/checkin, not thread-safe) | none (file-based; less pressure to pool, but nothing stops adding one) |

None of this is a defect in either driver — they're each idiomatic for their
own C library. It does mean tauorm's Core cannot assume "the driver hands
back a `Vec`-of-params exec + upfront rows" (true for Postgres, false for
SQLite's cursor model) or vice versa. Every dialect-specific behavior above
gets normalized behind one interface (`src/dbapi.tr`) so the expression
language, schema layer, and ORM above it are written once, against that
interface, the same way SQLAlchemy's own `Dialect`/`DBAPI` boundary lets one
`Query` implementation run against a dozen real databases.

Phase 1 (below) landed on an even more literal version of "normalized behind
one interface" than originally planned here: both dialects' `execute()`
returns the exact same concrete `QueryResultSet` class (not their own
result-set type each separately satisfying a generic interface) — see
"Status" for why, and for three real `tauraroc` limitations found building
this that shaped the decision.

## Backends are optional `taupkg` dependencies, not hard dependencies

Consumers pick their backend(s) at build time via `taupkg` features — the
same optional-dependency mechanism `taupkg` already supports (see
`taupkg/src/features.tr`), not something tauorm invents:

```toml
[deps]
postgres = { path = "../taupostgres", optional = true }
sqlite3  = { path = "../tausqlite3",  optional = true }

[features]
default  = ""
postgres = "postgres"
sqlite   = "sqlite3"
all      = "postgres, sqlite"
```

```
taupkg build --features postgres          # Postgres only
taupkg build --features sqlite            # SQLite only
taupkg build --features postgres,sqlite   # both
```

Core and the ORM (schema definition, the expression language, session/mapper
logic) compile standalone with **no** features enabled — only the two
`src/*_dialect.tr` adapter modules need a real driver present. Adding a
third backend later (MySQL, say) is "write one more dialect adapter +
declare one more optional dep", not a change to Core or the ORM.

## Architecture at a glance

Flat `src/`, matching `taupostgres`/`tausqlite3`'s own layout (both drivers
import each other's sibling modules by bare name — `from row import Row`,
`from statement import Statement` — not through a nested package path), not
the nested `core/`/`sql/`/`schema/`/`orm/` nesting an earlier draft of this
proposal sketched. Two concrete reasons, not just convention-matching:

- Tauraro's module resolution for a bare `from x import y` only reliably
  finds `y` as a **sibling** of the importing file, not a parent/ancestor
  directory — confirmed the hard way while building Phase 1 (`src/dialects/
  sqlite_dialect.tr` importing sibling-of-parent `src/core/db_error.tr`
  bare failed to resolve; a full dotted `from core.db_error import ...`
  was needed instead, and even that behaved inconsistently once a second
  entry point outside `src/` was involved). A flat layout means every
  intra-package import is a plain bare sibling import, no dotted paths, no
  resolution-depth surprises.
- It also avoids a real naming trap: `taupostgres`/`tausqlite3` already
  claim short top-level names like `connection.tr`, `result.tr`, `errors.tr`
  on `TAURARO_PATH` once tauorm depends on them. tauorm's own modules avoid
  those exact names (`db_error.tr` not `errors.tr`, `query_result.tr` not
  `result.tr` — the latter doubly so since `Result` is also Tauraro's
  built-in `Result[T,E]` name, the same collision taupostgres's own
  PROPOSAL.md flagged for `PgResult`).

```
tauorm/
  taupkg.toml            # optional postgres/sqlite3 deps, feature-gated (see above)
  src/
    tauorm.tr               # re-export hub -- `from tauorm import DbError, DbConnection, ...`
                              # Does NOT import either dialect adapter (see
                              # their own header comments) -- a consumer
                              # imports postgres_dialect.tr/sqlite_dialect.tr
                              # directly, whichever their build has installed.

    # ── Core: dialect boundary (Phase 1 -- DONE, see Status) ──────────────
    dbapi.tr                  # DbConnection interface -- execute/begin/commit/
                                # rollback/close. Non-generic (see Status for why).
    db_error.tr                 # DbError { code, message } -- the one error type
                                  # every dialect adapter raises, translated from
                                  # whatever the underlying driver actually threw.
    query_result.tr               # QueryResultSet -- ONE concrete, dialect-agnostic
                                    # materialized result (column names, rows, null
                                    # flags, affected-row count) + a step-cursor API
                                    # (next()/get_*()/is_null()) on top of it.
    postgres_dialect.tr             # PostgresConnection: wraps postgres.Connection,
                                      # materializes PgResult into QueryResultSet.
    sqlite_dialect.tr                 # SqliteConnection: wraps sqlite3.Connection,
                                        # steps its Statement to completion into
                                        # QueryResultSet.

    # ── planned, not yet built ─────────────────────────────────────────────
    engine.tr                # create_engine(url) -- picks a dialect by URL
                               # scheme, owns a connection pool
    async_engine.tr             # create_async_engine(url) + AsyncConnection
                                  # -- see "Sync and async" below
    pool.tr                        # thin wrapper unifying postgres.Pool with
                                     # a from-scratch pool for drivers (like
                                     # sqlite3) that don't have one

    # ── Phase 2: composable SQL expression language (planned) ─────────────
    expression.tr             # ColumnClause, BinaryExpression, and_/or_/not_,
                                # ==, !=, <, >, like(), in_(), is_(None)
    sql_functions.tr             # func.count/sum/avg/now/coalesce/... (dialect-
                                   # aware rendering, e.g. Postgres NOW() vs
                                   # SQLite datetime('now'))
    select_stmt.tr                  # select(*cols).where().join().group_by()
                                      # .order_by().limit()/.offset()
    insert_stmt.tr                     # insert(table).values(...); .returning()
                                         # where the dialect supports it
    update_stmt.tr                        # update(table).where().values(...)
    delete_stmt.tr                           # delete(table).where()
    sql_compiler.tr                             # Statement -> (sql_string, params)
                                                  # per-dialect; owns paramstyle
                                                  # rendering ($1.. vs ?)

    # ── Schema definition + DDL (planned) ──────────────────────────────────
    table.tr                  # Table(name, *columns, metadata=)
    column.tr                    # Column(name, type, primary_key=, nullable=,
                                   # default=, unique=, index=)
    col_types.tr                    # Integer, BigInteger, String(length), Text,
                                      # Boolean, Float, Numeric, Date, DateTime,
                                      # JSON, ... + per-dialect DDL rendering
    constraints.tr                     # PrimaryKeyConstraint, ForeignKey,
                                         # UniqueConstraint, CheckConstraint, Index
    metadata.tr                           # MetaData: create_all(engine) /
                                            # drop_all(engine)

    # ── ORM: mapping, sessions, relationships (planned) ────────────────────
    declarative.tr             # DeclarativeBase, mapped_column()
    mapper.tr                     # Mapper: class <-> Table binding + instance
                                    # instrumentation for the unit of work
    identity_map.tr                  # per-Session cache keyed by (class, PK)
    unit_of_work.tr                     # tracks new/dirty/deleted instances,
                                          # flushes in FK-safe topological order
    session.tr                             # Session: add/delete/get/query/
                                             # commit/rollback/flush
    async_session.tr                          # AsyncSession mirroring Session
    orm_query.tr                                 # select(Model).where(...) --
                                                   # compiles down to Core's select()
    relationship.tr                                 # one-to-many/many-to-one/
                                                      # one-to-one/many-to-many,
                                                      # lazy vs. joined loading

  example/app/              # planned: a CRUD + relationships walkthrough,
                              # runnable against either backend
  tests/                    # test_dbapi_sqlite.tr (runs for real, in-memory),
                              # test_dbapi_postgres.tr (needs a live server --
                              # see its header comment); NOT a single shared
                              # generic test function over both dialects --
                              # see Status for the compiler bug that forced
                              # duplicating the ~15 lines of test logic instead
  docs/                     # planned: 01-getting-started.md, 02-core.md,
                              # 03-orm-relationships.md
```

## Sync and async, from day one

Both a sync and an async story are in scope for v1 -- `Session`/`Engine`
alongside `AsyncSession`/`AsyncEngine` -- but "async" means different things
per backend, and the proposal is explicit about that rather than papering
over it:

- **Postgres**: `async_query.tr`'s `send_query`/`next_result` already exist,
  but taupostgres's own PROPOSAL.md documents that `next_result` still
  blocks on the socket the way `PQgetResult` does on a normal connection --
  not true non-blocking I/O yet (that needs `PQsocket` wired into an event
  loop, called out there as "a bigger, separate undertaking"). `AsyncEngine`
  for Postgres will use `send_query`/`next_result` as-is; it gets the async
  *shape* (so calling code composes with other `std.async` work) today, and
  a real event-loop-integrated non-blocking path becomes a drop-in
  improvement underneath the same interface later — not an API change.
- **SQLite**: the C API has no async story at all. `AsyncConnection` for the
  SQLite dialect offloads each blocking call onto `std.async`'s thread pool
  (the same shape aiosqlite/Diesel-async use for SQLite) rather than
  pretending it's non-blocking.

Both are hidden behind the same `DbConnection`/dialect interface Core uses,
so ORM code written against `AsyncSession` doesn't change when a dialect's
underlying async story improves.

## Status

**Phase 1 (Core dialect boundary) is done and verified for real, not just
compiled.** `dbapi.tr`'s `DbConnection` interface, `db_error.tr`'s `DbError`,
`query_result.tr`'s `QueryResultSet`, and both `postgres_dialect.tr`/
`sqlite_dialect.tr` adapters are written and check clean against their real
drivers. `tests/test_dbapi_sqlite.tr` actually runs (in-memory, no server)
and passes all 4 assertions: CREATE TABLE, INSERT (`rows_affected() == 1`),
SELECT round-tripping the inserted value back out, and a not-null check.
`tests/test_dbapi_postgres.tr` is written and `--check`-clean but untried
against a live server (none available while building this). Both `taupkg
install --features sqlite` and `taupkg build --features sqlite` succeed
end-to-end through the real package manager, not just via a manually-set
`TAURARO_PATH`.

Three real `tauraroc` limitations surfaced while building this, all worth
fixing upstream but none blocking Phase 1 once designed around:

1. **Segfault on mutually-referential generic bounds.** A generic function
   declared `[C: DbConnection[RS], RS: DbResultSet]` (one bound's type
   argument is another type parameter of the same function) crashes the
   compiler outright. Reproduced with a ~15-line repro unrelated to any DB
   code. This is *why* `DbConnection` ended up non-generic and both
   dialects converge on the single concrete `QueryResultSet` (see below)
   instead of each having their own result-set type satisfying a shared
   generic interface, the originally-planned design.
2. **No dynamic-dispatch boxing for bare interface return types.** A
   function declared `-> SomeInterface` (not a generic bound, the interface
   name used directly as a concrete return type) type-checks under
   `--check`, but real codegen fails: `incompatible types when returning
   type 'X *' but 'Y_obj' was expected`. So there's no cheap way to return
   "some type satisfying DbResultSet" without generics either.
3. **Mis-codegen for method calls on a concrete return type inside a
   monomorphized generic function body.** Even with a single, non-crashing
   generic bound (`[C: DbConnection]`), a generic function's body calling
   methods on `QueryResultSet` (a concrete type returned by the
   interface-bound `execute()`) emits bare, unqualified C calls (e.g.
   `next(sel_rs)`) instead of `QueryResultSet_next(sel_rs)`. This is why
   `tests/dbapi_shared.tr` (one shared generic test function called from
   both `test_dbapi_postgres.tr`/`test_dbapi_sqlite.tr`) was abandoned in
   favor of duplicating the ~15 lines of test logic per dialect.

Combined, these three pushed Phase 1 toward less genericity than planned:
one shared concrete `QueryResultSet` class instead of a generic
`DbResultSet` interface each dialect implements separately. This is a
reasonable trade for now (see `query_result.tr`'s header comment for the
one real cost: SQLite's step-cursor gets walked to completion inside
`execute()` rather than streamed, matching how Postgres's own `PgResult`
already works) and can be revisited once/if the compiler's generic-bound
handling matures.

Two non-compiler lessons from actually getting a real test passing, both
now documented in the adapter files' header comments so they aren't
rediscovered the same way:

- **A bare call to a `throws E` function (no `?`, no explicit
  `Result[T,E]` annotation) does not auto-unwrap or auto-raise — it tries
  to bind the raw `Result` struct to whatever type the variable was
  otherwise inferred as, and fails codegen with `incompatible types when
  initializing type 'X *' using type 'Result'`.** This reproduces
  identically whether the bare call is inside a plain function, inside
  `try:`, or inside `main()` — `taupostgres`'s own README example
  (`mut db = Connection.connect(...)`, no `?`) does not actually compile as
  literally written. The working idiom (confirmed via
  `tests/lang/05_error_handling.tr`'s "mixed models" section upstream):
  bind to an explicit `mut r: Result[T, E] = call(...)`, then check
  `r.is_err` / use `r.ok` / `r.err`. This is also *better* than `try:/
  except e:` for error translation, not just a workaround: `except e:`
  binds only the exception's stringified form (confirmed empirically —
  `e.sqlstate` on a caught `PgError` fails to compile), while `r.err` is
  the real typed error object, so `DbError.code` genuinely carries
  Postgres's SQLSTATE / SQLite's result code instead of being stuck at `""`.
- **`get_str`/`column_str`-style accessors return a *borrowed* view into
  the driver's own buffer, not an owned copy** (confirmed by reading
  `tausqlite3`'s implementation: a raw pointer cast, no allocation).
  Storing one in a `Vec[str]` across further `step()` calls (SQLite) or
  past `PgResult.clear()` (Postgres) reads back corrupted memory — hit this
  as a real, silent-until-runtime bug (`--check` and even a clean compile
  don't catch it) where a passing-looking build printed garbage instead of
  the inserted row. Fixed by forcing a copy (`+ ""`) at materialization
  time; see both dialect files' `_materialize`/`execute()` implementations.

## What's deferred, and why

- **Migrations** (an Alembic equivalent — versioned schema changes,
  autogenerate-by-diffing-metadata): explicitly out of scope for this
  proposal. `MetaData`/`Table`/`Column` (Phase 2 below) is the same
  introspectable schema representation a migration tool would diff against,
  so nothing here forecloses it — it's a separate, later package/phase the
  same way connection pooling started as "a separate `taupostgres-pool`
  package" for taupostgres before that call was reversed once it turned out
  to be little enough code to fold in directly. Migrations are a much bigger
  undertaking than that pooling case, so deferring is the more defensible
  default here, not just a smaller version of the same call.
- **Additional dialects** (MySQL/MariaDB, etc.): the dialect interface is
  designed for this from the start (see above), but no third driver exists
  in this workspace yet to build one against honestly.
- **Advanced ORM features** not in the initial phases: polymorphic
  inheritance mapping, hybrid properties, event hooks (`before_flush`, etc.),
  composite primary keys, save-update/delete cascades beyond the basic FK
  ordering Phase 3's unit of work needs. SQLAlchemy grew these over years;
  matching all of them in v1 would delay a working Core+ORM indefinitely.
  Track these as a backlog once Phases 1-4 are real and tested.

## Suggested sequencing

1. **Phase 1 — Core dialect boundary.** `dbapi.tr`, `db_error.tr`,
   `query_result.tr`, both dialect adapters: **done, see Status.**
   `engine.tr`'s `create_engine(url)` (pick a dialect by URL scheme, own a
   connection pool) is the one piece of the original Phase 1 scope not yet
   built — today a consumer constructs `PostgresConnection`/
   `SqliteConnection` directly rather than through a URL-dispatching
   `Engine`. Worth doing before Phase 2 starts consuming it, but the harder
   part (the dialect boundary itself, and proving it against a real,
   passing test) is the part that was actually uncertain.
2. **Phase 2 — Expression language + schema.** `select`/`insert`/`update`/
   `delete` builders, `Table`/`Column`/`types`/constraints, `MetaData.
   create_all`/`drop_all`. Success condition: the same Python-esque query
   built once compiles to correct, dialect-appropriate SQL + params for both
   backends.
3. **Phase 3 — ORM mapping + sync Session.** Declarative base, `Mapper`,
   identity map, a unit of work that handles single-table insert/update/
   delete ordering. No relationships yet.
4. **Phase 4 — Relationships.** one-to-many/many-to-one/one-to-one,
   many-to-many via association tables, lazy vs. joined loading.
5. **Phase 5 — Async.** `AsyncEngine`/`AsyncConnection`/`AsyncSession` per
   the "Sync and async" section above, exercised against both dialects.
6. **Phase 6 (backlog, not scheduled).** Migrations, additional dialects,
   the advanced ORM features listed under "What's deferred."

Each phase should land with tests run against *both* enabled features
(`taupkg test --features postgres` and `--features sqlite`) before moving
on — the two drivers' real divergence (table above) is exactly what would
let a Postgres-only assumption slip into Core silently otherwise.
