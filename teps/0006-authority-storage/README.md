---
tep: 0006
title: How the authority stores and changes data
status: draft
authors:
  - "@The127"
approvers:
  - "@tidegate-dev/maintainers"
tracking-issue: null
created: 2026-11-06
last-updated: 2026-11-06
replaces: []
superseded-by: []
---

# TEP-0006: How the authority stores and changes data

## Summary

The authority stores its data in Postgres. This TEP defines how the authority's logic, the code that decides who gets access, reads and writes that data: how it reads rows, changes them and writes them back, and what happens when two changes to the same row collide.

For each table there are two Go structs. One is generated from the table by a tool and uses the database driver's types, and this TEP calls it the database model. The other uses plain Go types, and this TEP calls it the code model. The logic only ever sees the code model.

A method that changes data doesn't run its own SQL. It puts each write on a list, which this TEP calls a unit of work (uow), and then calls `Commit`, which opens a transaction, runs the SQL for every write on the list and commits. Every `UPDATE` and `DELETE` in that transaction checks that the row is still at the version the method read. If it isn't, `Commit` returns an error, the API answers with the status code `aborted`, and the client reads the data again before it sends the change again.

Queries that don't return one table's rows, like a list with a join, return a struct shaped like the query result, which this TEP calls a read model.

## Motivation

The logic has to be easy to read and to test, and it must never lose a change. When it works with the database driver's types directly, three things get in the way of that:

- The logic fills up with database details. "This user has no local password" becomes a check for `NULL`, and "unknown user" becomes the driver's error for an empty result. Logic written like that only runs against a real Postgres, so even a test for a permission check needs a database.
- Nothing detects a lost update. When two admins change the same role at the same time, the second `UPDATE` overwrites the first, and nobody decided that. In a system that grants access, that's a security problem rather than a nuisance.
- The audit log has to record every change in the same transaction as the change itself, which means that some code has to know, before the transaction commits, exactly which rows it is about to write.

### Goals

- The logic uses no database type and gets no database error.
- Every change runs in one transaction, so either all of it is written or none of it.
- An update or delete of a row that changed since the logic read it fails instead of overwriting the row.
- Before a transaction commits, the code knows which rows it inserts, updates and deletes, so the audit log can record them in the same transaction.
- Queries with joins, lists and counts don't have to go through the code models.

### Non-goals

- How the audit log stores and protects its events. That's a TEP of its own, still to be written.
- Event sourcing, a second database for reads, and caching.
- Which tables exist and what they hold. That's TEP-0003, the data model.

## Proposal

This is how a method in the logic changes a user's password:

```go
func (s *Users) ChangePassword(ctx context.Context, id uuid.UUID, hash string) error {
    user, err := s.store.Users().ByID(ctx, id)
    if err != nil {
        return err
    }

    user.PasswordHash = &hash

    uow := s.store.NewUnit()
    uow.Users().Update(user)

    return uow.Commit(ctx)
}
```

`s` is the struct the method belongs to, and `s.store` is the field through which it reaches the database. That field holds the connection pool. `s.store.Users()` returns the queries for the users table, and `ByID` runs a `SELECT` and returns the user as a code model, together with the version of the row it read.

`s.store.NewUnit()` returns an empty unit of work. `uow.Users().Update(user)` puts the changed user on the unit's list of pending writes and runs nothing. `Commit` opens a transaction, runs an `UPDATE` whose `WHERE` clause names both the ID and the version, and commits. If another transaction changed the row in the meantime, the version no longer matches, the `UPDATE` affects no row, and `Commit` rolls back and returns `model.ErrConflict`.

The API answers `model.ErrConflict` with the status code `aborted`. Since nothing was written, the client can always retry by reading the data again and sending its change again.

### User stories

- An admin changes a role's permissions in the web UI while another admin changes the same role from the CLI. The second change fails with `aborted`, and the UI shows the role as it is now and asks the admin to apply the change again.
- A contributor writes the method that revokes a binding. The method reads the binding, checks that the caller may revoke it, puts the delete on the list and calls `Commit`. Its test uses a fake store and runs without Postgres.
- An auditor wants to know which rows an approval changed. The audit event lists them, because `Commit` knew every pending write when it wrote the event.

### Risks and mitigations

- A contributor forgets to call `Commit`, so the list is dropped and the change is lost without an error. Every method has a test that checks what is in the database afterwards, which makes this fail in CI.
- A contributor changes a field and forgets to call `Update`. The same tests catch it.
- A method reads a row after it put a write for that row on the list and gets the old data, because nothing is written before `Commit`. A method must not read a row it has a pending write for, and reviewers check for that.
- Some writes are only allowed while certain rows don't exist. The first user, for example, may only be created while the users table is empty. If two such writes run at the same time, both see that the rows don't exist, both write, and the condition is broken. The version check can't prevent this, because it compares the version of a row that was read, and here nothing was read. The section "Conditions that aren't a version check" says how a unique constraint or an advisory lock prevents it.

## Design details

### Packages

This design adds two packages. A check in CI enforces which code may import which:

- `model` holds the code models, the read models and the error values. It imports nothing.
- `store` runs the SQL. It converts between database models and code models, and it holds the unit of work. It's the only code that imports the database driver and the generated queries.
- The logic imports `model`, and it declares an interface for the store, so that its tests can use a fake.
- The API handlers import `model` and the logic.

### Two models per table

The database model is generated from the table and mirrors it with the driver's own types, such as the driver's type for `uuid` and a nullable type for `password_hash`. The code model lives in `model` and uses plain Go types:

```go
type User struct {
    ID           uuid.UUID
    Name         string
    PasswordHash *string // nil means no local password
    Version      uint32
}
```

`store` converts between the two, so that `NULL` becomes `nil`, a query with no result becomes `model.ErrNotFound`, and nothing outside `store` ever sees a driver type.

A code model holds no slices of other code models. A user doesn't carry their login tokens. A token carries its user's ID instead. A method that needs a user and their tokens reads the user with `ByID` and the tokens with a query for that user's ID, and a page that shows both at once gets a read model with a join, which returns them in one query. Each token is written through its own `Add`, `Update` and `Remove`, so `Update(user)` never touches a token. That keeps the code models small and avoids loading every token whenever a user is read.

The code that creates a code model also assigns its ID, so that a method can put a user and a token that points to that user on the list in the same unit. IDs are version 7 UUIDs, which start with the time, so the primary key index stays in insert order.

### The version

The version is Postgres's `xmin` column. Every table has it, it holds the ID of the transaction that last wrote the row, and it changes with every write to the row and never otherwise, so no table needs a version column of its own.

Every `INSERT` and `UPDATE` ends with `RETURNING xmin`, and `store` writes the result into the code model's `Version`. Every `UPDATE` and `DELETE` compares `xmin` with that `Version`:

```sql
UPDATE users SET name = $2, password_hash = $3 WHERE id = $1 AND xmin = $4 RETURNING xmin
```

`xmin` is 32 bits and wraps around after about four billion transactions. Since Postgres 9.4, freezing a row leaves `xmin` in place, so a row that isn't written keeps its version, and a stale `Version` can only match a row by accident if the code model is kept across four billion transactions. We accept that. A restore from a dump rewrites every row and so changes every version, and a client that kept a code model across a restore gets a conflict and reads the data again.

### The unit of work

For the users table, `store` defines two structs, `Users` and `UnitUsers`. Every other table gets the same pair, like `LoginTokens` and `UnitLoginTokens`.

```go
// Users queries the users table on the connection pool.
type Users struct { ... }

func (u *Users) ByID(ctx context.Context, id uuid.UUID) (*model.User, error)

// UnitUsers belongs to a unit of work. It queries like Users and records
// writes, which the unit runs at Commit.
type UnitUsers struct {
    Users
    unit *Unit
}

func (u *UnitUsers) Add(user *model.User)
func (u *UnitUsers) Update(user *model.User)
func (u *UnitUsers) Remove(user *model.User)
```

The store's `Users()` returns a `*Users`, and a unit's `Users()` returns the `*UnitUsers` that belongs to it. Both are methods, not fields, so a caller can't replace them.

```go
type Unit struct { ... }

func (s *Store) NewUnit() *Unit
func (u *Unit) Users() *UnitUsers
func (u *Unit) LoginTokens() *UnitLoginTokens
func (u *Unit) Commit(ctx context.Context) error
```

A unit of work holds a list of pending writes. `Add`, `Update` and `Remove` on `uow.Users()` and on the other per-table types append to that list and run no SQL. Reads always run on the connection pool right away, outside any transaction, whether they go through `s.store.Users()` or through `uow.Users()`, because the transaction only exists while `Commit` runs.

`Commit` opens that transaction and runs the SQL for each pending write in the order it was appended. For `Add` it runs an `INSERT` and writes the returned `xmin` into the code model's `Version`. For `Update` it runs an `UPDATE` of every column with `WHERE id = $1 AND xmin = $2` and writes the returned `xmin` into `Version` as well. For `Remove` it runs a `DELETE` with the same `WHERE` clause.

When the `UPDATE` or the `DELETE` matches no row, `store` runs a `SELECT` for the ID to find out why. If the row exists, its `xmin` changed, and `Commit` returns `model.ErrConflict`. If the row doesn't exist, `Commit` returns `model.ErrNotFound` for an update, while a delete counts as done, because the row is gone, which is what the method wanted. So a second `Remove` of the same row succeeds.

A unique constraint that refuses an `INSERT` or `UPDATE` is a conflict too, because it means another transaction used the value after the method checked that it was free, and `Commit` returns `model.ErrConflict` for it.

Once the audit log TEP says what an event holds, `Commit` writes the audit events for the list in the same transaction before it commits. Until then it only commits. Postgres aborts the whole transaction on the first error, so when one write fails, none of the writes is kept, and a unit can't be used again after `Commit`, whether it succeeded or not.

A method must not read a row it has a pending write for, because the read would return the old data. A method that has to read after writing calls `Commit` first and then creates a new unit.

### Writes that run several statements

Not every pending write is a single `INSERT`, `UPDATE` or `DELETE`. A per-table type can offer a method that appends a write which runs several statements inside `Commit`, and the caller can't tell the difference, so tests use the same fake store for both kinds.

Deleting a user's expired login tokens is one example. `uow.LoginTokens().RemoveExpired(userID)` appends a write that runs one `DELETE` for all matching rows at commit time. It reads no rows and checks no version, because it deletes whatever matches at that moment. The next section has the other example.

### Conditions that aren't a version check

The version check detects that a row changed after it was read. It can't detect that a row was inserted after a method checked that none existed, because there is no row whose version the method could have read. Such a condition is handled in one of two ways:

- With a unique constraint, when the condition is "no row with this value exists". The database checks it, and `Commit` returns `model.ErrConflict` when it fails.
- With an advisory lock, when no constraint can express the condition. Creating the first user is the example: when two operators create the first user at the same time, both could see an empty table and both could insert. For this case, `store` offers `uow.Users().AddFirst(user)`, which at commit time runs three statements in the transaction: `SELECT pg_advisory_xact_lock(...)`, a `SELECT` that checks that the users table is empty, and the `INSERT`. Postgres releases the lock when the transaction ends. If the table isn't empty, `Commit` rolls back and returns `model.ErrUserExists`.

Only `store` takes advisory locks, and a method that does so has a comment that names the condition it protects.

### Read models

A query that doesn't return one table's rows returns its own struct in `model`:

```go
tokens, err := s.store.LoginTokens().ActiveForUser(ctx, userID) // []model.ActiveLoginToken
```

The tool that generates the database models also generates such a struct for every query. A read model has no `Version` and can't be passed to `Add`, `Update` or `Remove`, and the query behind it can use any SQL.

### Errors

`model` defines `ErrNotFound`, `ErrConflict` and the errors of methods like `AddFirst`, such as `ErrUserExists`. The logic returns them as they are or wrapped, and the API handlers map `ErrNotFound` to the status code `not_found` and `ErrConflict` to `aborted`, which the [gRPC status codes](https://grpc.io/docs/guides/status-codes/) define for a failed optimistic concurrency check. Every other error reaches the client as `internal` with a generic message.

## Security considerations

- Trust boundaries: none move. Only the authority connects to Postgres, as TEP-0002 says.
- Privilege: none granted. The version check prevents a lost update. Without it, two concurrent changes to a role or a binding could widen access without anyone deciding so.
- Credentials and secrets: code models hold password and token hashes, never the plaintext. A code model with a secret field has no `String` method, so it can't be printed into a log by accident. The audit events hold the table, the ID and the kind of write, no column values.
- Audit: `Commit` knows every write before it commits, which is what lets the audit log record every change in the same transaction. Whether an event can be skipped or forged is for the audit log TEP.
- Failure behavior: a conflict, a lost connection or a failed audit write rolls the transaction back. No partial change reaches the database. A failed change never grants anything.
- Threat model: a client that sends the same change twice at once gets one success and one `aborted`. This design doesn't protect against a compromised authority replica, because the version check runs inside the replica. Postgres hardening is the operator's job, as TEP-0004 says.

## Compatibility and upgrade

The change is internal to the authority and needs no migration, because every Postgres table has `xmin`. During a rolling upgrade, a build that doesn't check versions can run next to one that does. When the old build writes a row, the new build's `UPDATE` of that row matches nothing, and the client retries.

## Test plan

- The logic: unit tests with a fake store, for every method and for the errors it maps. They run without Postgres.
- `store`: integration tests against Postgres. For each table: writing a code model and reading it back, `ErrNotFound`, a conflict between two units on `Update` and on `Remove`, an `Update` of a deleted row, a repeated `Remove` that succeeds, a unique violation that returns `ErrConflict`, and that `Commit` writes nothing when one write fails. For `AddFirst`: two units that wait for the lock, and only one succeeds.
- The API handlers: one test that `ErrConflict` reaches the client as `aborted`.

## Rollout

A first change adds `model`, `store` and the unit of work for every table and moves the tests of the logic to a fake store. Every table added later follows the same pattern. No feature flag, because nothing is user-facing.

## Drawbacks

- Every table costs a code model and the conversion in `store`, next to the generated database model. That's a few dozen lines per table.
- Forgetting `Commit` or `Update` compiles. Tests catch it, the compiler doesn't.
- A method can't read a row it has a pending write for. Most methods don't need to. The ones that do call `Commit` twice.
- Reads run outside the transaction, so a method can read two rows from different moments. The version check catches that only when the method writes one of them.

## Alternatives

- Keep the generated types in the logic and convert only the IDs. It's the least code, but it solves none of the three problems in the motivation. The `NULL` and empty-result checks stay in the logic and keep its tests on Postgres, nothing detects a lost update, and no code knows what a transaction writes, so the audit log has nothing to record. It also ties every signature in the logic to the generated code, which changes whenever a column or a query does.
- Define the code models next to the logic and let `store` import them from there. This works, but then `store` can call the logic, and nothing stops a contributor from doing that. A `model` package that imports nothing keeps the two apart.
- Run each write at once, with `Begin` opening the transaction. A method can then read what it wrote, and locks work the usual way. The cost is a transaction that stays open while the method runs, and a `Rollback` that every method has to defer.
- A callback instead of a unit, like `store.Do(ctx, func(tx) error)`. Nobody can forget `Commit`, but a method with more than two steps becomes a closure inside a closure.
- Code models with slices of other code models, like a user with their tokens. Loading a user then loads the tokens too, and `Update` has to work out which tokens changed. Both are hard to get right.
- A `version` column instead of `xmin`, counted up on every `UPDATE` and checked by a trigger, so the database refuses a stale write by itself. It survives a dump and restore and works on other databases. It costs a column and a trigger per table, and the authority only runs on Postgres.
- `SELECT ... FOR UPDATE` on every read. Simple, but it blocks readers that never write, and every method holds its locks until it commits.
- An ORM. It generates the SQL at run time, so a reviewer can't see what runs against the database. For a system whose database decides access, reviewers have to see the SQL.

## Unresolved questions

- How the list of pending writes becomes audit events, and whether a write like `RemoveExpired` produces one event or one per row. That's for the audit log TEP, but it has to be settled before the audit log is built.

## Implementation history

- 2026-11-06: Draft opened in [#16](https://github.com/tidegate-dev/enhancements/pull/16).
