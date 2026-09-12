# query-builder-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

SQL, built as a value and rendered as bytes.  `SELECT`, `INSERT`,
`UPDATE` and `DELETE`, with joins, CTEs, set operations, upsert and a
where-clause expression tree — for SQLite, PostgreSQL, MySQL and
sql-engine-nv's own dialect, from one tree.

It does not execute anything.  `std.sql`, pager-nv's driver, or
whatever speaks to your database takes the bytes and the parameter list
and runs them.

Five modules, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **builder** | `qbbuild` | you are writing a query |
| the **tree** | `qbast` | you are inspecting or transforming one |
| the **render** | `qbrender` | you are about to execute one |
| the **dialects** | `qbdialect` | you are porting, or writing SQL by hand |
| the **faults** | `qberror` | a statement would not render |

## Adding it, and checking it

```bash
novo pkg add query-builder-nv    # into your novo.toml
novo pkg build                   # type- and effect-check the package
novo test --isolate tests/qbrender_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: query-builder-nv.<module>.<fn>`.
They turn green one at a time as bodies land.

## The one example that will work

```novo
use qbast
use qbbuild
use qbrender

fn search(term: Str, min_age: Int) -> Str [io]
    var q = qbbuild.from(qbbuild.columns(qbbuild.select(), ["id", "name"]),
                         "users", "u")
    q = qbbuild.and_where(q, qbbuild.like("name", term))
    if min_age > 0
        q = qbbuild.and_where(q, qbbuild.ge("age", qbbuild.int_value(min_age)))
    q = qbbuild.order_by(q, "created_at", true)
    q = qbbuild.page(q, 0, 25)

    match qbrender.render_str(QbStmtSelect(q), qbrender.options(QbPostgres))
        Err(f) => ""
        Ok(r)  =>
            // r.sql    — SELECT id, name FROM users AS u
            //            WHERE name LIKE $1 AND age >= $2
            //            ORDER BY created_at DESC LIMIT 25 OFFSET 0
            // r.params — [QbText(term), QbInt(min_age)]
            r.sql
```

Change `QbPostgres` to `QbSqlite` and the same tree renders `?` twice,
with the parameter list in the same order.

## The load-bearing interface

`qbast.QbBind` — a value in the tree, at the point it is used.

```novo norun:pseudo
pub enum QbExpr
    …
    QbBind(value: QbValue)
```

There is **no API in this package that takes a parameter list**.  The
list comes out of `qbrender.render`, beside the bytes, in the order the
placeholders appear.

That is the whole reason to use a query builder rather than joining
strings, and it is the thing sqlx's `QueryBuilder` gets right with
`push_bind` and is explicit about.  The alternative — the caller writes
`?` into a fragment and appends to a list — has a failure mode with no
error message: the string and the list agree today, somebody edits the
`WHERE` clause tomorrow, and the query binds the tenant id to the row
limit.  Every row of another customer's data comes back, and nothing
anywhere says so.

Here that state has no spelling.  `qbbuild.eq("id", v)` builds
`QbCmp(QbEq, QbCol("id"), QbBind(v))`; the ordinary comparison binds by
construction, and a caller has to reach for `QbLit` or `QbRaw` — both
of which carry a warning in their own doc comment — to put a value in
the text at all.

The second decision the rest follows from is that **a dialect is a
table, not a branch**.  Every place the four disagree is a function in
`qbdialect` answering one fact — the placeholder, the quote character,
whether `RETURNING` exists, whether a backslash escapes inside a string
— and `qbrender` reads them.  Forty `match d` sites is how a renderer
ends up supporting three dialects properly and the fourth almost, with
nothing saying which sites were missed.

## What cannot be built, and must be written by hand

`qbrender.cannot_build()` answers this list at run time; here it is in
prose.

- **DDL.** `CREATE`, `ALTER`, `DROP`, `GRANT`. A builder that builds
  DDL is a migration tool, and a migration tool wants a different shape
  — an ordered, named, reversible list of changes — than a query
  builder does.
- **Window functions.** `OVER (PARTITION BY … ORDER BY … ROWS …)` is a
  sub-grammar with frames, exclusions and named windows;
  sql-engine-nv's `WindowCall` and `Frame` show the size of it. Use
  `QbRaw` for the whole `SELECT` item.
- **`GROUPING SETS`, `CUBE`, `ROLLUP`.**
- **`VALUES` as a table source**, `LATERAL`, `TABLESAMPLE`, and table
  functions.
- **Locking beyond `FOR UPDATE`** — no `FOR SHARE`, `SKIP LOCKED` or
  `NOWAIT`.
- **Vendor hints** (`/*+ … */`), and comments generally: sql-engine-nv
  does not lex `--` or `/* */` at all, so a rendered comment would not
  merely be ignored there, it would fail to parse.
- **Type names.** `QbCast` writes the type as given and translates
  nothing: `TEXT` is not `VARCHAR` is not `STRING`, and a builder that
  guessed would be wrong about exactly the columns that matter.
- **Scope checking.** A correlated subquery is expressible and
  unchecked — nothing here knows whether the outer table the inner
  query names is in scope. The database will say.

## What each dialect cannot say

`qbrender.unsupported(stmt, dialect)` answers this for a given
statement, one fault per feature, so a port is a list rather than a
search.

| | sql-engine | SQLite | PostgreSQL | MySQL |
| --- | --- | --- | --- | --- |
| quoted identifiers | **no** | yes | yes | yes (backtick) |
| parameters | **no** | `?` | `$1` | `?` |
| comments | **no** | yes | yes | yes |
| `RETURNING` | no | yes | yes | **no** |
| `ILIKE` | no | no | yes | no |
| `DISTINCT ON` | no | no | yes | no |
| `NULLS FIRST/LAST` | no | yes | yes | **no** |
| `FULL OUTER JOIN` | no | yes | yes | **no** |
| conflict target | no | yes | yes | **no** |
| `FOR UPDATE` | no | **no** | yes | yes |
| `TRUE` / `FALSE` | no | no | yes | no |
| `IS DISTINCT FROM` | no | yes | yes | `<=>`, inverted |

Two of sql-engine-nv's "no"s are worth reading twice, because they are
what reading its dialect turned up rather than what anyone expected:

- **It cannot quote an identifier at all.** Its lexer recognises
  neither `"name"` nor a backtick, so an identifier is a run of ASCII
  letters, digits and underscores and nothing else. A column named
  after a reserved word cannot be spelled, and this package refuses it
  — `QbUnquotableIdent` — rather than emitting SQL that lexes as two
  tokens where one was meant.
- **It has no placeholder token.** `?` is not in its punctuation set,
  although pager-nv's `driver.execute` takes a `params` list beside the
  SQL. So a statement with a `QbBind` cannot be executed by
  sql-engine-nv as written today; `qbdialect.supports(QbSqlEngine,
  QbFeatParameters)` answers false and the fault names it. That is a
  gap in sql-engine-nv rather than in this package, and it is stated
  here so nobody discovers it at execution time.

## The three refusals worth knowing before you build a query

**A join with no `ON` is refused.** `QbJoinWithoutOn`, for every kind
but `QbCross`. An inner join with no condition is a cross join written
by accident, and on two tables of a million rows it is the single most
expensive typo in SQL. Writing `QbCross` is how a caller says they
meant it.

**`IN ()` is refused.** `QbEmptyInList`. The expression is a syntax
error in all four dialects, and the two plausible repairs — render
`FALSE`, or drop the clause — return opposite row sets. Choosing one
for the caller would be choosing which rows they get, silently, in the
case where the list came from a filter that matched nothing.

**An `INSERT` with no column list is refused.** `QbInsertNoColumns`. An
`INSERT` without one depends on the table's declaration order, which a
builder cannot see and which changes the day somebody adds a column.

And one that is **not** refused: an `UPDATE` or `DELETE` with no
`WHERE`. "Set every row's flag" is a real statement. `qbrender.unguarded`
is the question, `QbRenderOptions.require_where` is the switch, and a
program that runs user-driven queries turns it on.

## The escape hatch, and how to find it

`QbRaw(sql)` splices text in exactly.  Nothing parses it, escapes it or
checks it, and a caller that builds one out of input has written the
injection this package exists to prevent.

So it is countable: `qbast.raw_count` answers how many a statement has,
and `QbRenderOptions.forbid_raw` refuses a statement containing any.
A build that wants to prove no query in the program was assembled by
hand asserts the second.

## Why `core`, and why no dependencies

Nothing here reads or writes anything: the render appends to a buffer
the caller owns.  That is what makes the whole package testable as
bytes, with no database in the room.

The obvious dependency would be sql-engine-nv — its `cell.Cell` as the
bound-value type, its `sqlast` as the tree.  Both are refused for one
reason: a query builder that pulls in a whole SQL engine to construct a
string is a builder nobody uses against PostgreSQL, and three of the
four dialects here are not sql-engine-nv.  `QbValue` is six variants
and the conversion to any driver's value type is one `match`.

The trees are different anyway.  `sqlast` is a **parser's** output: it
has every statement SQLite accepts, including the DDL this package does
not build, and it has no notion of a bound parameter because its
dialect has no placeholder token.  `qbast` is a **builder's** input:
four statements, and `QbBind` in the tree.

No device claim.  A statement is a tree of lists of strings; a device
that talks to a database talks to it over a network, and the query it
sends was built on the other end.

## Ports

[SeaQuery](https://github.com/SeaQL/sea-query),
[sqlx](https://github.com/launchbadge/sqlx)'s `QueryBuilder` and
[SQLAlchemy Core](https://docs.sqlalchemy.org/en/20/core/) are the
reference implementations.  SeaQuery's dialect split is the model for
`qbdialect`; sqlx's `push_bind` is the model for `QbBind`;
SQLAlchemy Core's separation of the statement from the connection is
the model for having no execution here at all.

## Licence

Apache-2.0.
