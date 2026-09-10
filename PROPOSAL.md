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

Phase 1 (below) is built exactly as described here: each dialect's
`execute()` returns its OWN result-set type (`PostgresResultSet`/
`SqliteResultSet`), separately satisfying the shared `DbResultSet`
interface, generic over `DbConnection[RS]`. Getting there took an
intermediate detour through a single shared, eagerly-materialized result
class after hitting real `tauraroc` generics/interfaces bugs — see
"Status" for that history and the bugs involved, all since fixed upstream.

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
  those exact names (`db_error.tr` not `errors.tr` — doubly motivated since
  `Result` is also Tauraro's built-in `Result[T,E]` name, the same
  collision taupostgres's own PROPOSAL.md flagged for `PgResult`).

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
    dbapi.tr                  # DbResultSet interface (column_count/column_name/
                                # next/get_*/is_null/rows_affected/release) +
                                # generic DbConnection[RS] (execute/begin/commit/
                                # rollback/release). RS is unbounded on the
                                # INTERFACE declaration itself (see Status for
                                # why) -- the RS: DbResultSet constraint is
                                # enforced at every generic use site instead.
    db_error.tr                 # DbError { code, message } -- the one error type
                                  # every dialect adapter raises, translated from
                                  # whatever the underlying driver actually threw.
    postgres_dialect.tr            # PostgresResultSet (implements DbResultSet,
                                     # walks an already-materialized PgResult) +
                                     # PostgresConnection (implements
                                     # DbConnection[PostgresResultSet]).
    sqlite_dialect.tr                # SqliteResultSet (implements DbResultSet,
                                       # a thin wrapper over Statement's own
                                       # step-cursor) + SqliteConnection
                                       # (implements DbConnection[SqliteResultSet]).
    engine.tr                          # Engine[C: DbConnection[RS], RS: DbResultSet]
                                         # -- a thin wrapper around one open
                                         # connection (execute/begin/commit/
                                         # rollback/release). No URL-based
                                         # create_engine(url) dispatch (see its
                                         # own header comment for why) and no
                                         # pooling yet (see "planned" below).

    # ── planned, not yet built ─────────────────────────────────────────────
    async_engine.tr             # create_async_engine(url) + AsyncConnection
                                  # -- see "Sync and async" below
    pool.tr                        # thin wrapper unifying postgres.Pool with
                                     # a from-scratch pool for drivers (like
                                     # sqlite3) that don't have one

    # ── Phase 2: SQL expression language (Phase 2 -- DONE, see Status) ────
    expression.tr              # SqlExpr: col()/lit()/eq()/ne()/lt()/gt()/le()/
                                 # ge()/like()/and_()/or_()/not_()/is_null()/
                                 # is_not_null(), built via explicit constructor
                                 # functions (not operator overloading -- see
                                 # this file's header comment for why) +
                                 # .render(dialect, params) -> dialect-correct
                                 # SQL text, collecting bound params in order.
    sql_compiler.tr               # SqlCompiled { sql, params } -- what every
                                    # statement builder below compiles to,
                                    # ready for Engine.execute(sql, params).
    select_stmt.tr                  # select_from(table).cols([...]).where(expr)
                                      # .limit(n).compile(dialect). Named
                                      # select_from(), not select() -- see its
                                      # own header comment (a real Windows
                                      # winsock2.h collision, not a Tauraro one).
    insert_stmt.tr                     # insert(table).values([names],[values])
                                         # .compile(dialect)
    update_stmt.tr                        # update(table).set([names],[values])
                                            # .where(expr).compile(dialect)
    delete_stmt.tr                           # delete(table).where(expr)
                                               # .compile(dialect)

    # ── Schema definition + DDL (Phase 2 -- DONE, see Status) ──────────────
    table.tr                  # Table(name, Vec[Column]) -- .col(name),
                                # .create_sql(dialect), .drop_sql()
    column.tr                    # Column: name/type/primary_key/nullable/
                                   # unique + builder chaining (.pk()/.not_null()
                                   # /.is_unique()) + .ddl(dialect)
    col_types.tr                    # Integer/BigInteger/Text/VarChar(n)/Boolean/
                                      # Float/DateTime, each with per-dialect DDL
                                      # rendering (e.g. Integer+primary_key ->
                                      # SERIAL on Postgres, INTEGER on SQLite)
    metadata.tr                        # MetaData (Vec[Table] registry) +
                                         # create_all/drop_all -- generic FREE
                                         # functions (not methods; see this
                                         # file's header comment for why),
                                         # calling .next() once per DDL
                                         # statement the same way
                                         # dbapi_shared.tr's tests do

    # ── planned (not built this phase) ─────────────────────────────────────
    sql_functions.tr           # func.count/sum/avg/now/coalesce/... (dialect-
                                 # aware rendering, e.g. Postgres NOW() vs
                                 # SQLite datetime('now'))
    constraints.tr                # PrimaryKeyConstraint, ForeignKey,
                                    # UniqueConstraint, CheckConstraint, Index
                                    # (today: primary_key/nullable/unique live
                                    # directly on Column; joins/foreign keys
                                    # aren't needed until relationships, a
                                    # later ORM phase)

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
  tests/                    # dbapi_shared.tr's run_shared_tests[C: DbConnection[RS],
                              # RS: DbResultSet] is ONE generic function, called
                              # unmodified from both test_dbapi_sqlite.tr (runs for
                              # real, in-memory) and test_dbapi_postgres.tr (needs a
                              # live server -- see its header comment). test_engine_
                              # sqlite.tr proves Engine[C,RS] the same way.
                              # test_phase2_sqlite.tr runs a full CREATE TABLE / INSERT
                              # / SELECT / UPDATE / DELETE round trip through the
                              # schema layer + query builders + Engine, in-memory,
                              # asserting both the compiled SQL text AND the actual
                              # row data at every step (11 assertions, all passing).
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
compiled, with the originally-planned generic design intact.** `dbapi.tr`'s
`DbResultSet` interface and generic `DbConnection[RS]`, `db_error.tr`'s
`DbError`, and both `postgres_dialect.tr`/`sqlite_dialect.tr` adapters
(each with its OWN streaming result-set class -- `PostgresResultSet`/
`SqliteResultSet` -- satisfying `DbResultSet`, not a single shared
materialized class) are written and check clean against their real
drivers. `tests/dbapi_shared.tr`'s `run_shared_tests[C: DbConnection[RS],
RS: DbResultSet]` is ONE generic function, run unmodified against both
dialects from their own thin entry files. `tests/test_dbapi_sqlite.tr`
actually runs (in-memory, no server) and passes all 4 assertions: CREATE
TABLE, INSERT (`rows_affected() == 1`), SELECT round-tripping the inserted
value back out, and a not-null check. `tests/test_dbapi_postgres.tr` is
written and `--check`-clean but untried against a live server (none
available while building this). Both `taupkg install --features sqlite`
and `taupkg build --features sqlite` succeed end-to-end through the real
package manager, not just via a manually-set `TAURARO_PATH`.

This took two passes. The first hit three real `tauraroc` limitations
(a segfault on mutually-referential generic bounds, missing dynamic-
dispatch boxing for bare interface return types, and mis-codegen for
method calls inside a monomorphized generic body calling through an
interface-bound receiver) and fell back to a single shared, eagerly-
materialized `QueryResultSet` class to work around all three at once.
Those three (plus two more found integrating the fix: multi-type-arg
generic call parsing/inference, and a non-generic class implementing a
generic interface with concrete type arguments silently discarding them)
were then fixed upstream in `tauraroc` itself, so the second pass restored
the original per-dialect-streaming-class design and deleted the
materialization workaround entirely.

Two smaller things surfaced integrating the fixed compiler, both worked
around rather than blocking on another fix cycle:

- **A bound on a generic INTERFACE's own type parameter** (`interface
  DbConnection[RS: DbResultSet]:`, as opposed to a bound on a generic
  *function's* parameter, which works fine) **hangs `tauraroc --check`
  indefinitely.** Reproduced with a 4-line, DB-unrelated repro. Worked
  around by declaring `DbConnection[RS]` unbounded in `dbapi.tr` and
  keeping the `RS: DbResultSet` constraint only at every generic function/
  class use site (`run_shared_tests[C: DbConnection[RS], RS:
  DbResultSet]`), which is unaffected and works correctly.
- **The interface-vtable-wrapper codegen emits a method's plain name
  instead of its C-keyword-escaped name.** `close()` legitimately compiles
  to `_tr_fn_close` everywhere else in the generated C (it collides with
  POSIX `close(fd)`), but the generated vtable initializer referenced the
  plain, non-existent `SqliteConnection_close`, an undefined-reference link
  failure. Sidestepped by naming the interface method `release()` instead
  of `close()` throughout `dbapi.tr` and both dialect adapters -- not a
  general fix (any interface method colliding with a C keyword/stdlib name
  would hit the same issue), but sufficient here.

**`engine.tr` (`Engine[C, RS]`, a thin generic wrapper around a single
DbConnection) is also done**, closing out the rest of Phase 1's original
scope (see "Suggested sequencing" below for what's still explicitly not
in it -- pooling). Building it surfaced 4 MORE real `tauraroc` bugs, all
specific to generic CLASSES with two cross-referencing type parameters
(`class Engine[C: DbConnection[RS], RS: DbResultSet]:`) -- generic
FUNCTIONS with the identical bounds, which is all `dbapi_shared.tr` uses,
were unaffected throughout. Unlike the two bugs just above, these four
were fixed at the root in `tauraroc`'s own source (`~/tauraro`), not
worked around:

1. A CLASS's own bound-satisfaction check for a second, cross-referenced
   type parameter only resolved correctly when that parameter's name was
   a single character -- `_is_type_param_in_scope` checked the enclosing
   FUNCTION's generics but never the enclosing CLASS's, falling back to a
   `strlen == 1` heuristic that happened to catch single-letter names by
   coincidence. Fixed by also checking `self.classes.get(current_class_name)
   .generics` in `sema.tr`.
2. The monomorphized struct name was mangled two different ways in
   different places (`Engine_MyConn_MyRS` vs
   `Engine_MyConn_ptr_MyRS_ptr`) for the exact same instantiation --
   `synth_class_suffix` (used to name a declared-type lookup, e.g. a local
   variable's own type) lacked the "strip a trailing `*` without
   appending `_ptr`" special-case `type_args_suffix` (used for a
   constructor call's own explicit type args) already had. Fixed by
   adding the matching special-case to `synth_class_suffix` in
   `codegen/c.tr`.
3. A generic method declared `throws E -> T` didn't re-wrap its `return`
   into a `Result` at the monomorphized call site -- returned the raw
   pointer where every caller expected a `Result` struct. `ensure_mono`'s
   method-body-generation loop was the one place in the whole codegen that
   never set `cur_throws_ty` before generating a method body, unlike every
   other such loop. Fixed by setting/restoring it there too.
4. Accessing a generic-typed FIELD from outside its class (e.g.
   `engine.conn.execute(...)`) fell back to using the literal type
   parameter name as a function prefix (`C_execute`) instead of the
   resolved concrete type. The `EPropAccess` (field access) case in
   `sema.tr` never substituted the field's generic type through the
   object's own concrete type args, unlike the adjacent `EMethodCall`
   case, which already did. Fixed by applying the same
   `_subst_ret_generics` call there.

All four were verified with minimal, DB-unrelated repros before and after
the fix, then verified STABLE via a gen1->gen2->gen3 self-hosting
fixpoint: `tauraroc` (gen1, with the fixes) compiled itself to produce
gen2, gen2 compiled itself to produce gen3, and gen2/gen3 emit
byte-identical generated C for the same input (the gen1->gen2 and
gen2->gen3 *binaries* differ byte-for-byte, expected for a compiler with
no dedicated reproducible-build engineering -- e.g. embedded timestamps --
but their actual compilation behavior has demonstrably converged). The
fixed `tauraroc.exe` was then deployed as the machine's canonical
compiler (old binary kept as a `.bak-<timestamp>` alongside it, matching
this project's own established convention for that).

One smaller inference gap surfaced integrating `Engine`, worked around
rather than needing a fifth compiler fix: a bare, un-bracketed static
factory call (`Engine.wrap(conn)`) relying on inference from the
assignment target's type annotation didn't emit a matching
`Engine_wrap` declaration reachable from the call site
(`implicit declaration of function 'Engine_wrap'`). Explicit type
arguments on the call (`Engine[SqliteConnection, SqliteResultSet].wrap
(conn)`, as `tests/test_engine_sqlite.tr` does) sidestep it entirely.

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
  `tausqlite3`'s implementation: a raw pointer cast, no allocation), valid
  only until the buffer is reused or freed. Hit this as a real,
  silent-until-runtime bug (`--check` and even a clean compile don't catch
  it): an early version stored `column_str()`'s result in a `Vec[str]`
  across further `step()` calls, and read back garbage on the next access
  since SQLite had already reused that buffer for the new row.
  `SqliteResultSet.get_str()`/`column_name()` force a copy (`+ ""`) for
  this reason. `PostgresResultSet` doesn't need the same defensive copy:
  taupostgres's `PgResult` keeps every row's buffer alive until
  `clear()`/`release()`, a wider validity window than SQLite's per-step
  reuse, and nothing in the current design stores a value across a call
  boundary the way the abandoned materialization approach did.
- **A statement with no result rows (CREATE TABLE, INSERT, ...) does not
  actually execute in SQLite until `next()`/`step()` is called at least
  once** -- `execute()` only prepares and binds (see `sqlite_dialect.tr`).
  Hit this as a real bug: an early version of `dbapi_shared.tr` never
  called `.next()` on a CREATE TABLE's result, so the table silently never
  got created, and the very next INSERT failed with "no such table". Fixed
  by calling `.next()` once on every `execute()` result before use,
  CREATE/INSERT included -- Postgres already executes inside `execute()`
  itself, so the same call there is a harmless no-op, making this the
  correct portable idiom rather than a SQLite-only workaround.

**Phase 2 (SQL expression language + schema) is done and verified for
real** -- `test_phase2_sqlite.tr` runs a full CREATE TABLE / INSERT /
SELECT / UPDATE / DELETE round trip in-memory, asserting both the exact
compiled SQL text and the actual row data at every step (11 assertions,
all passing), and both dialect paths `--check` clean. Two more real
`tauraroc` bugs surfaced, both the SAME class of bug as one already fixed
in `ensure_mono` for class methods (see above) -- `ensure_mono_func_n`
(the NEWER machinery for monomorphizing a generic FREE function with N
type params) turned out to have the identical gap, just never exercised
by Phase 1's generic functions since none of them both threw AND took an
`Engine[C, RS]`-shaped argument:

- A monomorphized generic free function's C signature used its bare
  success type instead of `Result` when the function was `throws`-declared
  (`gen_func_sig`'s own check for this was correctly applied elsewhere,
  but `ensure_mono_func_n` computed its signature independently and
  skipped it) -- fixed by adding the same `f.throws_ty.name != ""` check.
- The same monomorphization path never set `cur_throws_ty` before
  generating the function body either (identical to the class-method bug
  already fixed) -- fixed the same way, set/restore around
  `gen_func_body`.

Both reproduced concretely via `metadata.tr`'s `create_all` (a generic
free function taking `engine: Engine[C, RS]`, declared `throws DbError`)
called with explicit type arguments (`create_all[SqliteConnection,
SqliteResultSet](md, engine, dialect)`) -- exactly the pattern
`Engine.wrap`'s own inference gap (see above) already established needs
explicit type args for this class of call. `create_all`/`drop_all` return
`bool` rather than being void-only `throws` functions, sidestepping a
separate, narrower version of the same underlying gap (an implicit-void
generic free function's return-wrapping) without needing a third
compiler change for what was, by that point, a clearly-understood root
cause.

One naming collision, unrelated to any of the above: a top-level `pub def
select(table: Table) -> Select:` collides with Windows' `winsock2.h`
`select()` (the socket multiplexing syscall) at the C level, since the
generated runtime pulls that header in for its own networking support and
Tauraro's keyword-escaping covers language keywords, not arbitrary
platform-library symbol names. Renamed to `select_from()`; see
`select_stmt.tr`'s header comment.

This session's self-hosting verification was lighter for this batch of
fixes than Phase 1's full gen1->gen2->gen3 fixpoint: gen1 (with the fix)
successfully self-compiled to gen2, confirming the fix doesn't break
self-hosting, without repeating the full byte-identical-codegen check a
second time. Several build attempts during this session hit transient,
environment-level failures (a lingering `tauraroc.exe`/`gcc.exe`/`cc1.exe`
process from an earlier, timed-out attempt racing a fresh one over the
same `build/` directory, and once a build that was still genuinely
in-progress past a 5-minute wait rather than actually stuck) -- not
compiler bugs; resolved by killing stale processes and/or simply waiting
longer before concluding a run had failed.

**Phase 3 (declarative mapping + Session) is done and verified for
real** -- `test_mapped_sqlite.tr` defines a plain `@mapped class User:`
(zero hand-written row-mapping code), then exercises `Session[User,
SqliteConnection, SqliteResultSet]`'s identity map and unit of work
(`add`/`get`/`mark_dirty`/`delete`/`commit`) through a full add-insert,
get-round-trip, identity-map-hit-on-repeat-get, update, delete, and
unrelated-row-survives-delete sequence against real in-memory SQLite (9
assertions, all passing).

The design took a real detour first. The user corrected an earlier
(wrong) assessment that Tauraro has no macro system and therefore no way
to support declarative mapping — it has a genuine one (`macro def
name(item) -> code:`, see `~/tauraro/docs/lang/advanced/10_macros.md`): a
compile-time function that receives a reflection of the class it's
attached to (`item.fields` — name+type pairs — for a class) and returns
Tauraro source text the compiler parses and splices in before sema, so
generated code is fully type/borrow-checked like anything else. `@mapped`
(`mapped.tr`) uses exactly this to generate `__table()`/`from_row()`/
`dump()`/`key_of()` from a class's plain fields.

Getting from "the macro system exists" to "a working generic
`Mapper[T]`/`Session[T]`" surfaced **7 more real `tauraroc` bugs** this
session (in addition to the 4 fixed closing out Phase 1/2, see above), all
found via minimal, ORM-unrelated repros before touching tauorm code, and
all fixed at the root in `~/tauraro/src/{sema.tr,codegen/c.tr}`, verified
via a gen1->gen2->gen3 self-hosting fixpoint (byte-identical generated C)
and the full `tests/lang`+`tests/regression` suite (no regressions; 3
pre-existing, unrelated failures — `fmt` idempotency on two example files,
one `cdylib` test — reproduced identically against the untouched original
compiler, so not something introduced here). In the order found:

1. A generic METHOD whose own `[T]` merely repeats its enclosing class's
   OWN generic parameter (`class Bag[T]: ... def add[T](self, obj: T):`,
   the pattern the generics docs require) was wrongly routed through the
   machinery meant for a method with a genuinely independent generic on an
   otherwise non-generic class (`Wrapper.echo[T]`), emitting a bogus
   `<Cls>_<method>__MONO_<targ>` forward declaration typed `Cls* self` --
   `Cls` being the bare, un-monomorphized name, undeclared. Fixed by
   skipping that path whenever the class itself is already generic (the
   class-level monomorphization already covers those methods correctly).
2. `T.static_method(args)` through a generic type parameter (`T` in
   `def build[T](n: int) -> T: return T.make(n)`) miscompiled to a call to
   an undeclared global function (`make(T, n)`, "T" undeclared) --
   `class_name` resolution for a static call fell back to nothing for a
   bare, in-scope type-parameter receiver. Fixed by resolving it through
   the active `type_subst` substitution at the two places that recover a
   static call's class name and its static-vs-instance check.
3. `T()` (default-constructing a generic type parameter to its
   eventual concrete type) was rejected outright at sema ("name 'T' is
   not defined"), never reaching codegen. Fixed by recognizing an in-scope
   type parameter as a legitimate bare name and by resolving it through
   `type_subst` in construction codegen.
4. `ClassName.method` referenced as a bare VALUE (no immediate call --
   e.g. assigned to a `def(...)->R` local, or passed as a callback
   argument) miscompiled to `ClassName->method`, as if `ClassName` held an
   instance -- the immediate-call form (`ClassName.method(args)`) was
   already correct via a separate path and unaffected. Fixed by adding
   sema lowering for this case to a proper qualified-function-value
   reference, plus codegen wrapping it into the same `TrFnVal` struct an
   ordinary top-level function reference already gets.
5. `obj.field.method()` chained off a duck-typed, generic-parameter-typed
   receiver (`obj: T` inside a generic function) silently dropped the
   `.method()` call, returning the raw field where the method's result was
   expected (e.g. `obj.n.to_str()` compiling to plain `obj->n`, an `int`
   where a `TrStr` was expected). Fixed the "unresolved property-access
   type" codegen fallback to also resolve through `type_subst` before
   giving up, not just when the type was already empty.
6. A "callable FIELD call" (`obj.f(args)` where `f: def(...)->T` is a
   FIELD, not a method — Mapper[T]'s `load`/`dump`/`key_of` fields'
   exact shape) never substituted the field's declared (always-generic)
   return type through the receiver's own concrete type args, unlike the
   adjacent method-call case a few lines above, which already did.
   Reproduced directly (`h.maker(42)` on a `Holder[Foo]` emitting `T* f =
   ...`, "unknown type name 'T'"). Fixed by applying the same
   `_subst_ret_generics` substitution there.
7. A local variable assigned from a monomorphized generic class's OWN
   method call (`mut f = h.build(42)`, `h: Holder[Foo]`, `build(self, n:
   int) -> T`) was auto-dropped at scope exit using the RECEIVER's
   monomorphized drop function (`_trdrop_Holder_Foo`) instead of its OWN
   type's (`_trdrop_Foo`) -- undefined reference, a link error. Root
   cause: a fallback specifically meant for a "factory" shape (`mut h =
   Holder[int].make(5)`, where the call's OWN result type genuinely IS the
   same generic class as the receiver, just missing its type args in the
   HIR) didn't check that the result and receiver types actually matched
   before applying, so it fired just as wrongly for `h.build(42)`
   returning an unrelated type. Fixed by adding that check.

One more was found but deliberately NOT fixed, in favor of a clean
design workaround: a macro-generated PLAIN TOP-LEVEL `pub def` (spliced
outside any `extend` block) gets its prototype correctly declared and
call sites resolve fine (`--check` passes), but its function BODY is
never emitted into any compiled `.c` file -- an undefined-reference link
error, reproduced with a minimal macro unrelated to tauorm. Root-causing
this would mean tracing how `expand_macros`-spliced declarations relate
to the resolver's own per-module decl tracking (built BEFORE macro
expansion runs), a materially bigger and less contained change than the
seven above. `@mapped` sidesteps it entirely by generating only
`extend`-block STATIC methods (`User.from_row`, `User.dump`, `User.
key_of`, referenced directly as bare `def(...)->R` values via fix #4
above) rather than top-level wrapper functions -- `extend`-block macro
output was already proven solid by tauraro's own `@derive_eq`/
`@derive_clone` tests, so this isn't a workaround so much as staying on
the well-tested path.

`Mapper[T]`/`Session[T]`'s design is a direct consequence of what's
and isn't possible here: Tauraro's generics have no way to construct a
generic `T()` or call `T.static_method()` through a bare, unbound type
parameter (bug #3 above is the closest it gets, and that's a hard error,
by design — not something a future fix should change, since there's no
notion of "T must be default-constructible" a caller could opt into). A
classic SQLAlchemy-style `Session` that internally does `T.from_row(row)`
or `T()` for an arbitrary mapped type is therefore not implementable as
written; `Mapper[T]` instead carries `load`/`dump`/`key_of` as `def(...)
->R` VALUES, each a `@mapped`-generated static method referenced directly
(fix #4's mechanism) — fully generic and reusable across any `@mapped`
entity, without Session ever needing to construct or call through a bare
`T`.

Scope is deliberately narrow, matching a real v1: single-table entities
only (no relationships — Phase 4), a single-column integer `id` primary
key by convention (Django/Rails do the same; Tauraro's macro `item`
reflection exposes a field's name+type only, not per-field decorators, so
there's no way to read an explicit `@pk` annotation — see `mapped.tr`'s
header comment), and EXPLICIT dirty tracking (`session.mark_dirty(obj)`)
rather than SQLAlchemy's transparent attribute-change interception, which
Tauraro has no hook (`__setattr__`-equivalent) to implement honestly.

**Phase 4 (relationships) is done and verified for real** --
`test_relationship_sqlite.tr` defines `User`/`Post` (both `@mapped`,
`Post.user_id` an int FK column), seeds one user with two of their posts
plus a third post pointing at a never-inserted user id, then exercises
both directions against real in-memory SQLite: `load_many` returns
exactly the two real posts (not "everything"), `load_one` resolves a
post's author, and `load_one` on the dangling FK correctly fails with
`DbError("NOT_FOUND", ...)` (6 assertions, all passing). `column.tr`
gained `.references(table, column)` so a Table can document a foreign
key in its own DDL (`REFERENCES table(col)`) -- purely descriptive;
SQLite doesn't enforce it without an explicit `PRAGMA`, and neither
`load_many` nor `load_one` depend on it to work.

There is deliberately no SQLAlchemy-style lazy relationship PROXY (no
`user.posts` attribute that transparently queries on first access) --
same reasoning as Session's explicit dirty tracking: Tauraro has no
`__getattr__`-equivalent hook to intercept a plain field read. Both
directions are a single explicit call instead (`relationship.tr`'s
header comment spells out when to reach for `load_one` vs. an existing
`Session.get()` for the same read).

This surfaced the session's **8th real `tauraroc` bug, the most serious
one found** -- not a compile error but silent memory corruption at
runtime, no diagnostic at all. Calling a `def(...)->R` FIELD (e.g.
`Mapper[T].load: def(DbResultSet) -> T`, called as `mapper.load(rs)`
where `rs`'s static type is a generic parameter `RS` monomorphized to a
concrete class implementing the interface `DbResultSet`) derived the
indirect call's C function-pointer CAST from the ARGUMENT's own type
(`SqliteResultSet*`, 8 bytes) instead of the FIELD's actually-declared
parameter type (the interface, boxed as `DbResultSet_obj`, a 16-byte
`{vtable, data}` struct) -- a `TrFnVal`-indirect call is one unchecked
`void*`-typed cast, so the mismatched arity/size compiled cleanly and
corrupted the stack at the call. Root-caused with a minimal, DB-unrelated
repro (an `interface Shape`/`class Square implements Shape` pair, no
tauorm code at all) before touching anything in tauorm. Fixed in
`codegen/c.tr`'s indirect-call code path: the cast signature now prefers
the callee's own declared parameter types (only where they resolve, via
the active `type_subst`, to an actual interface name -- otherwise the
argument's own type is still used, unchanged, since that's correct for
every non-interface case), and arguments are boxed into that interface
(`SqliteResultSet_as_DbResultSet(rs)`) the same way an ordinary
non-indirect call's arguments already were.

Fixing that surfaced two smaller, adjacent issues, both fixed alongside it:

- The interface-vtable wrap function itself (`SqliteResultSet_as_
  DbResultSet`) is emitted once per (non-generic class, interface) pair,
  but that emission pass ran AFTER monomorphized generic function bodies
  in the overall output -- a monomorphized body calling the wrap function
  (as fix #8 now does) hit it before it was declared, an implicit
  wrong-type declaration and a real, loud compile error (not a crash;
  reordering was needed to get to the crash fix above at all). Fixed by
  moving that emission earlier, before monomorphized bodies, in
  `codegen/c.tr`'s top-level `generate()`.
- The wrap function is *only* ever generated for a class with an
  EXPLICIT `implements InterfaceName` clause -- a class satisfying an
  interface purely STRUCTURALLY (the normal way a generic bound like
  `RS: DbResultSet` is satisfied throughout this project, deliberately,
  since Phase 1) never gets one, because nothing records that
  relationship for this pass to find. This is existing, working-as-
  designed behavior, not a bug -- but `SqliteResultSet`/
  `PostgresResultSet` need it now that `Mapper[T].load` boxes into
  `DbResultSet`, so both dialect adapters were given an explicit
  `implements DbResultSet` (previously implicit/structural only); see
  `sqlite_dialect.tr`'s header comment.

All three (crash fix + reordering + explicit `implements`) were verified
together via the same gen1->gen2->gen3 self-hosting fixpoint and full
test-suite run as every other fix this session (no regressions; same 3
pre-existing, unrelated failures as always), then the compiler was
re-blessed.

**Phase 5 (async) is done and verified for real** -- `test_async_sqlite.tr`
round-trips real rows through real in-memory SQLite entirely via `await`
(insert, get, identity-map hit vs. a genuine miss, update, delete, an
unrelated row surviving another row's delete, and a raw standalone query),
8 assertions, all passing. `async_engine.tr`/`async_session.tr` ship the
CONCRETE-`async-def`-per-dialect pattern their own header comments already
called for (a generic `async def engine_execute[C,RS](...)` is still
blocked on the pre-existing, documented, not-root-caused bug: `await
generic_fn[T](...)`'s own expression type still resolves wrong -- verified
still open this session with the same minimal repro as before, unchanged
by anything fixed below).

Getting `test_async_sqlite.tr` to actually compile and run (not just
`--check`-clean) surfaced two more real `tauraroc` bugs, both found via
minimal, tauorm-unrelated repros before editing anything here, both fixed
at the root in `~/tauraro/src/codegen/c.tr`:

- **`await` on a `throws`-declared function produced C that tried to cast
  a `Result` struct to/from an integer** ("aggregate value used where an
  integer was expected" / "conversion to non-scalar type requested"),
  reproduced directly with a 5-line, DB-unrelated repro (`async def f()
  throws str -> int`, `await f(...)`). This one turned out to already be
  fixed on `master` (`emit_async_wrapper_for_call`/`gen_await_call` both
  heap-allocate-and-box `Result`/`Option`/`Tuple` returns instead of
  pointer-casting them) -- the machine's deployed `tauraroc.exe` binary
  simply predated that fix being rebuilt and blessed. Rebuilding from
  current `master` alone resolved it; no source change was needed for
  this one.
- **A monomorphized generic class's OWN struct definition gets corrupted
  when one of its fields is typed as ANOTHER not-yet-monomorphized
  generic class** (`Session[T,C,RS]`'s `engine: Engine[C,RS]` and
  `mapper: Mapper[T]` fields, hit whenever something textually before
  `main()`'s own variable declarations -- here, the concrete top-level
  `async def commit_users(sess: Session[...])`/`sqlite_execute(engine:
  Engine[...], ...)` wrapper functions the design above calls for --
  forces `Session`'s monomorphization to happen before `Engine`'s/
  `Mapper`'s own. Root cause: `ensure_mono`'s struct-body field loop
  calls `type_to_c(field.ty)` while already mid-way through writing this
  class's own `typedef struct { ... }` text into `mono_buf`; when a
  field's type is itself an unmonomorphized generic class, `type_to_c`
  recursively calls `ensure_mono` for IT, which writes ITS complete
  struct+prototypes through the exact same `mono_buf` redirect -- landing
  spliced into the middle of the outer struct's still-open field list
  (confirmed directly: `Engine`'s and `Mapper`'s entire struct
  definitions appeared interleaved between two of `Session`'s field
  lines in the generated header, a real, loud compile error, "has no
  member named 'mapper'" etc.). Fixed by pre-resolving every field's C
  type (triggering any nested `ensure_mono` calls to completion) BEFORE
  writing this class's own forward declaration and struct body, so a
  nested struct's text can only ever land strictly before the outer
  one's, never inside it. Reproduced and verified independently of
  tauorm with a minimal two-generic-class repro (a `B[X]` holding an
  `A[int]` field, `A` monomorphized in field position before any
  variable of type `A[int]` exists).

Both were verified via the same gen-to-gen self-hosting fixpoint
convention as every other fix in this document (compiled generated C
diffed byte-identical between two consecutive self-compiles) and the
full `tests/lang`+`tests/regression` suite (no regressions; same 3
pre-existing, unrelated failures as always -- `fmt` idempotency on two
example files, one `cdylib` test), then the compiler was re-blessed.

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

1. **Phase 1 — Core dialect boundary + Engine.** `dbapi.tr`, `db_error.tr`,
   both dialect adapters (each with its own `DbResultSet`-implementing
   class), and `engine.tr`'s `Engine[C, RS]`: **done, see Status.** No
   URL-based `create_engine(url)` dispatching by scheme, deliberately --
   see `engine.tr`'s header comment: Tauraro has no conditional
   compilation, so a function importing both dialect adapters to dispatch
   between them would force every consumer to install both optional
   backends regardless of which taupkg feature they actually enabled. Each
   dialect wraps its own connection factory directly instead
   (`Engine[SqliteConnection, SqliteResultSet].wrap(SqliteConnection.
   open(path)?)`).
2. **Phase 2 — Expression language + schema.** `select_from`/`insert`/
   `update`/`delete` builders, `Table`/`Column`/`col_types`, `MetaData.
   create_all`/`drop_all`: **done, see Status.** Verified end-to-end against
   SQLite (11 assertions, `test_phase2_sqlite.tr`), `--check`-clean for
   Postgres. Not yet done: `sql_functions.tr` (func.count/sum/now/...),
   dedicated constraint objects beyond what `Column` itself carries
   (foreign keys aren't needed until relationships, joins aren't needed
   until then either) -- reasonable to pick up alongside Phase 4
   (relationships) when there's a concrete need driving their design,
   rather than speculatively now.
3. **Phase 3 — ORM mapping + sync Session.** `@mapped` declarative
   entities, `Mapper[T]`, an identity map, and a unit of work handling
   single-table insert/update/delete ordering: **done, see Status.**
   No relationships yet.
4. **Phase 4 — Relationships.** one-to-many (`load_many`) and many-to-one
   (`load_one`) between `@mapped` entities, plus FK column metadata
   (`Column.references()`): **done, see Status.** One-to-one is the same
   `load_one` call against a column with a `.is_unique()` constraint, no
   new code needed. Many-to-many via association tables and joined
   (as opposed to explicit N+1) loading are not built -- no concrete need
   has driven their design yet; picking them up alongside real usage
   (rather than speculatively) matches how Phase 2 deferred FK/join
   support until Phase 4 actually needed it.
5. **Phase 5 — Async.** `async_engine.tr`/`async_session.tr` per the "Sync
   and async" section above: **done, see Status.** Verified end-to-end
   against SQLite (8 assertions, `test_async_sqlite.tr`); no dedicated
   Postgres async test, matching Phase 3/4's precedent (the dialect-
   agnostic logic being exercised -- here, `await` itself plus the
   concrete wrapper pattern -- was already proven dialect-agnostic in
   Phase 1, so only SQLite needs its own runnable test; Postgres has no
   live server available to actually run against here regardless).
6. **Phase 6 (backlog, not scheduled).** Migrations, additional dialects,
   the advanced ORM features listed under "What's deferred."

Each phase should land with tests run against *both* enabled features
(`taupkg test --features postgres` and `--features sqlite`) before moving
on — the two drivers' real divergence (table above) is exactly what would
let a Postgres-only assumption slip into Core silently otherwise.
