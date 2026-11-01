# skink-tegu

Database-agnostic ORM for Skink. One API over every relational driver in the ecosystem — PostgreSQL, MySQL/MariaDB, and SQL Server via `skink-dbx` wire drivers, plus SQLite through `std/db`.

```
scales add markoxley/skink-tegu
```

## Features

- **Dialect layer** (`tegu_dialect`) — placeholder styles (`?`, `$n`, `@Pn`), identifier quoting, type names, `LIMIT`/`OFFSET`/`TOP` spelling, auto-pk strategy (`RETURNING` / `OUTPUT INSERTED` / `last_insert_rowid()` / `LAST_INSERT_ID()`).
- **Unified connection** (`tegu_conn`) — one `Conn`/`Tx` surface over dbx drivers and SQLite; `?`-style args everywhere.
- **Model mapping** (`tegu_model`) — reflect-driven struct↔table: `Create`, `Drop`, `Insert` (auto-pk write-back), `Get`, `Save`, `Delete`, `All`, `Where`, `LoadRelated` (has-many eager load).
- **Fluent query builder** (`tegu_query`) — `SELECT … JOIN … WHERE … GROUP BY … HAVING … ORDER BY … LIMIT/OFFSET`, `Update`, `Delete`. Write `?` placeholders; they are rewritten per dialect and `?`s inside string literals are left alone.
- **Migrations** (`tegu_migrate`) — ordered, versioned, idempotent; each migration runs in its own transaction where the engine supports transactional DDL.

## Quick start

```skink
import "mod/markoxley/skink-tegu"
import "mod/markoxley/skink-tegu/tegu_model"
import "std/reflect"

pub struct User {
    [db(pk,auto)]
    id:   int64
    [db(notnull)]
    name: string
    age:  int
}

fn main() {
    // postgres://… | mysql://… | mssql://… | :memory: | file.db
    c, err := tegu.Open("postgres://user:pass@host/db")
    users, _ := tegu.ModelOf(&c, reflect.TypeOf<User>(User{}))

    tegu_model.Create(&users)

    u := User{name: "ada", age: 36}
    tegu_model.Insert(&users, int64(&u))   // u.id is populated

    var got User
    if tegu_model.Get(&users, u.id, int64(&got)) {
        println(got.name)
    }

    // transactions
    tx, _ := tegu.Begin(&c)
    txu, _ := tegu.ModelOf(&tx, reflect.TypeOf<User>(User{}))
    tegu_model.Insert(&txu, int64(&u))
    tegu.Commit(&tx)   // or tegu.Rollback(&tx)

    tegu.Close(&c)
}
```

Field tags attach to the line _above_ the field:

| Tag                                | Effect                                             |
| ---------------------------------- | -------------------------------------------------- |
| `[db(pk)]`                         | primary key (a field named `id` is the default pk) |
| `[db(auto)]`                       | auto-increment pk; value written back on Insert    |
| `[db(name=x)]`, `[column(name=x)]` | rename the column                                  |
| `[db(ignore)]`                     | skip the field                                     |
| `[db(notnull)]`                    | `NOT NULL` constraint in `Create`                  |

Supported field types: `int`, `int64`, `string`, `float`, `bool`.

## Query builder + joins

```skink
import "mod/markoxley/skink-tegu"
import "mod/markoxley/skink-tegu/tegu_query"
import "mod/markoxley/skink-dbx/dbx_value"

b := tegu.Select(&c, "users")
tegu_query.Join(&b, "LEFT JOIN", "posts p", "p.user_id = users.id")
tegu_query.ColsRaw(&b, "users.*, count(p.id) AS post_count")
tegu_query.Where(&b, "users.age > ?", []dbx_value.Value{tegu.Int(int64(18))})
tegu_query.GroupBy(&b, "users.id")
tegu_query.Having(&b, "count(p.id) > ?", []dbx_value.Value{tegu.Int(int64(2))})
tegu_query.OrderBy(&b, "users.id ASC")
tegu_query.Limit(&b, 10)
rows, err := tegu_query.Rows(&b)
```

## Eager loading

```skink
posts, _ := tegu.ModelOf(&c, reflect.TypeOf<Post>(Post{}))
rel := tegu_model.LoadRelated(&users, int64(&u), posts, "user_id")
var p Post
for tegu_model.Next(&rel, int64(&p)) { /* u.id's posts */ }
```

## Migrations

```skink
import "mod/markoxley/skink-tegu"
import "mod/markoxley/skink-tegu/tegu_migrate"

migs := []tegu_migrate.Migration{
    tegu.Mig(int64(1), "create widgets",
        "CREATE TABLE widgets (id BIGINT PRIMARY KEY, name TEXT)",
        "DROP TABLE widgets"),
    tegu.Mig(int64(2), "add color",
        "ALTER TABLE widgets ADD color TEXT", ""),
}
n, err := tegu.Migrate(&c, migs)   // applied count
v := tegu_migrate.Version(&c)      // highest applied version
tegu_migrate.Rollback(&c, migs, 1) // undo latest
```

Applied versions are recorded in a `_schema_migrations` table; `Migrate` is idempotent.

## Testing

- `bash run_tests.sh` — unit tests (dialect SQL rendering, model CRUD, joins, migrations) on real SQLite in-memory; the live test self-skips.
- Live: `SKINK_ORM_DSN=<dsn> ./live_test_bin` — see `run_tests.sh` output naming, or `test_live.sh` flows in sibling driver repos for Podman recipes (`postgres:16-alpine`, `mysql:8.0`, `mcr.microsoft.com/mssql/server:2022-latest`).
- `bash install_local.sh` — publish to a local `SCALES_GITHUB_MIRROR` and verify a consumer can `scales add` + build.
