# query-builder-nv

A query builder assembles a SQL statement as a value rather than by
joining strings, and hands the values a statement uses to the driver
separately from its text. This package builds `SELECT`, `INSERT`,
`UPDATE` and `DELETE`, with joins, common table expressions, set
operations, upserts and a where-clause expression tree, and renders
one tree into four dialects: SQLite, PostgreSQL, MySQL, and the subset
[sql-engine-nv](https://novo-lang.org/packages/sql-engine-nv) reads.
It executes nothing. The shape follows
[SeaQuery](https://github.com/SeaQL/sea-query),
[sqlx](https://github.com/launchbadge/sqlx)'s `QueryBuilder` and
[SQLAlchemy Core](https://docs.sqlalchemy.org/en/20/core/).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What it is

A **statement** here is a tree. `qbbuild.select()` answers an empty
one, and each call adds one fact to it and answers a new statement, so
a query assembled from three request filters, two of which are absent,
is a loop rather than a string concatenation.

A **bound value** is a value the driver sends beside the statement
rather than inside its text. On the wire it is written as a
**placeholder**: `?` in SQLite and MySQL, `$1` and `$2` in PostgreSQL.
**The value lives in the tree, at the point it is used.**
`qbbuild.eq("id", v)` builds a comparison whose right-hand side is the
value itself, and `qbrender.render` assigns the placeholder positions
as it walks. There is no function in this package that takes a
parameter list. The list comes out of the render, in the order the
placeholders appear.

That is the whole reason to use a builder. The alternative is a caller
writing `?` into a fragment and appending to a list kept beside it.
The string and the list agree today, somebody edits the `WHERE` clause
tomorrow, and the query binds the tenant identifier to the row limit.
Every row of another customer's data comes back and nothing anywhere
says so.

A **dialect** is a table of facts, not a branch. Every place the four
disagree is one function in `qbdialect` answering one question, and
the renderer reads them. `qbdialect.supports` is total over the
dialects and the features, so the table has a row per fact and each
row is testable on its own.

**Rendering is validating.** There is no separate checking pass: the
walk that emits the bytes is the walk that finds the join with no `ON`
clause. `qbrender.render` stops at the first fault, which is what a
program executing a query wants. `qbrender.check` is the same walk
collecting every fault, which is what a linter wants.

Nothing here reads or writes anything. A render appends to a buffer
the caller owns.

These are the render options and their defaults.

| Option | Default | What it does |
| --- | --- | --- |
| `pretty` | false | Newlines and indentation between clauses, for a log a person reads |
| `quote_all` | false | Quote every identifier rather than only the ones that need it |
| `require_where` | false | Refuse an `UPDATE` or `DELETE` with no `WHERE` |
| `forbid_raw` | false | Refuse a statement containing spliced text |
| `max_depth` | 64 | Refuse nesting deeper than this |
| `max_params` | 65,535 | Refuse more bound values than this, which is PostgreSQL's wire limit |

## Install

```
novo pkg add query-builder-nv
```

## Example

```novo
use std.list
use qbbuild
use qberror
use qbrender

fn main() [io]
    // SELECT id, name FROM users AS u
    var q = qbbuild.from(qbbuild.columns(qbbuild.select(), ["id", "name"]), "users", "u")

    // Each condition carries its value. Nothing is written into the text.
    q = qbbuild.and_where(q, qbbuild.like("name", "ada%"))
    q = qbbuild.and_where(q, qbbuild.ge("age", qbbuild.int_value(18)))

    // ORDER BY created_at DESC, then the first page of 25 rows.
    q = qbbuild.order_by(q, "created_at", true)
    q = qbbuild.page(q, 0, 25)

    // Render for PostgreSQL. The same tree renders `?` for SQLite,
    // with the values in the same order.
    match qbrender.render_str(QbStmtSelect(q), qbrender.options(QbPostgres))
        Err(f) => println(qberror.message(f))
        Ok(r)  =>
            // `r.sql` goes to the driver, `r.params` goes beside it.
            println(r.sql)
            println("${list.len(r.params)} parameters")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: query-builder-nv.<module>.<fn>` panic. The tests are
the specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `qbast` | The whole statement vocabulary as one set of types, and the five questions a caller can ask of a statement without rendering it. |
| `qbbuild` | One call per fact added: the comparisons, the four statements, their clauses, and the two shorthands `page` and `count_of`. |
| `qbrender` | The statement out as bytes with the values beside them, the options, the fault collector, and the two identifier and literal writers. |
| `qbdialect` | The four dialects, the twenty features they differ over, and the table that answers which has which. |
| `qberror` | Why a statement will not render, and the split between a statement that is wrong everywhere and one this dialect cannot say. |

The whole vocabulary is in one module because the tree is cyclic and
modules are not: a select holds expressions in its `WHERE`, and an
expression holds a select in `IN (…)` and `EXISTS (…)`.

## How to choose an entry point

**`qbrender.render_str` answers a `Str` and the values.** It is the
ordinary call.

**`qbrender.render` appends to a buffer the caller owns**, for a
program building a batch of statements into one buffer.
`qbrender.render_len` counts the bytes without writing them.

**`qbbuild.and_where` is the one to use, and `where_` is beneath it.**
`where_` replaces the condition. A builder with only `where_` makes
every caller write the combining match by hand, and the caller who
forgets replaces the tenant filter with the search filter.

**`qbrender.check` collects every fault**, where `render` stops at the
first. **`qbrender.unsupported` collects only the faults that are this
dialect's limits**, which is the list a port has to read.
**`qbrender.unguarded`** answers whether a statement is an `UPDATE` or
`DELETE` with no `WHERE`.

**`qbast.binds_of`, `raw_count`, `tables_of`, `target_table`,
`is_write` and `depth`** answer questions about a statement without
rendering it, for a program that logs, audits or routes queries.

## The rules a user needs

1. **The parameter list comes out of the render, never into it.** Hand
   `QbRender.params` to the driver exactly as it is. Reordering it is
   how a query binds the tenant identifier to the row limit.
2. **The ordinary comparison binds by construction.**
   `qbbuild.eq`, `ne`, `lt`, `gt`, `le`, `ge`, `in_values` and
   `between` all take a value and put it in the tree. Putting a value
   into the text takes `QbLit` or `QbRaw`, and a caller has to reach
   for one.
3. **`QbRaw` splices text in exactly.** Nothing parses it, escapes it
   or checks it, and a caller that builds one out of input has written
   the injection this package exists to prevent. It is countable:
   `qbast.raw_count` answers how many a statement has, and
   `forbid_raw` refuses a statement with any.
4. **A join other than a cross join must have an `ON` clause.**
   `QbJoinWithoutOn` refuses one that does not. An inner join with no
   condition is a cross join written by accident, and on two tables of
   a million rows it is the most expensive typo in SQL. `QbCross` is
   how a caller says they meant it.
5. **`IN ()` is refused.** `QbEmptyInList`. It is a syntax error in
   all four dialects, and the two plausible repairs — render `FALSE`,
   or drop the clause — return opposite row sets. Choosing one would
   be choosing which rows the caller gets in exactly the case where
   the list came from a filter that matched nothing.
6. **An `INSERT` must name its columns.** `QbInsertNoColumns`. An
   `INSERT` without a column list depends on the table's declaration
   order, which this package cannot see and which changes the day
   somebody adds a column.
7. **An `UPDATE` or `DELETE` with no `WHERE` is allowed.** Setting
   every row's flag is a real statement. `qbrender.unguarded` is the
   question and `require_where` is the switch a program running
   user-driven queries turns on.
8. **A set operation's arms may not carry their own `ORDER BY` or
   `LIMIT`.** `QbSetArmOrdered` refuses it in every dialect. SQLite
   allows it and PostgreSQL does not, and the parentheses that would
   make it portable change what the clause applies to.
9. **`QbCast` writes the type name as given.** `TEXT` is not
   `VARCHAR` is not `STRING`, and a builder that guessed would be
   wrong about exactly the columns that matter.
10. **Nothing checks scope.** A correlated subquery is expressible and
    unchecked. Whether the outer table the inner query names is in
    scope is the database's answer.
11. **An identifier containing the dialect's own quote character, or a
    zero byte, is refused everywhere.** `QbUnwritableIdent`. Doubling
    the quote is the correct escape and this package does it, but
    there is no escape for a zero byte.
12. **The sql-engine dialect cannot quote an identifier at all.** Its
    lexer recognises neither `"name"` nor a backtick, so an identifier
    is a run of ASCII letters, digits and underscores. A column named
    after a reserved word is refused with `QbUnquotableIdent` rather
    than rendered as SQL that lexes as two tokens where one was meant.
13. **The sql-engine dialect has no placeholder token.** A statement
    carrying a bound value cannot be executed by it as written today.
    `qbdialect.supports(QbSqlEngine, QbFeatParameters)` answers false
    and the render says so, rather than letting it be discovered at
    execution time.
14. **MySQL has no conflict target.** An upsert uses whichever unique
    index was hit, and there is no way to name one. A statement that
    named a target renders without it, and `qbrender.unsupported` says
    so.
15. **MySQL spells `IS DISTINCT FROM` as `<=>`, with the opposite
    sense.** The renderer writes it correctly and inverts the
    comparison. The feature row is there so a caller can see that a
    translation happened.
16. **SQLite needs a `LIMIT` in front of an `OFFSET`.** The renderer
    writes `LIMIT -1` for a bare offset.
17. **`RIGHT JOIN` needs SQLite 3.39 or later.** This package cannot
    see a server's version, so the SQLite row says the feature is
    present.

These are the twelve differences that decide whether a statement can
be rendered for a given dialect. `qbrender.unsupported` answers the
list for a statement.

| Feature | sql-engine | SQLite | PostgreSQL | MySQL |
| --- | --- | --- | --- | --- |
| Quoted identifiers | no | yes | yes | yes, with a backtick |
| Parameter placeholders | no | `?` | `$1` | `?` |
| Comments | no | yes | yes | yes |
| `RETURNING` | no | yes | yes | no |
| `ILIKE` | no | no | yes | no |
| `DISTINCT ON` | no | no | yes | no |
| `NULLS FIRST` and `NULLS LAST` | no | yes | yes | no |
| `FULL OUTER JOIN` | no | yes | yes | no |
| A conflict target on an upsert | no | yes | yes | no |
| `FOR UPDATE` | no | no | yes | yes |
| `TRUE` and `FALSE` as literals | no | no | yes | no |
| `IS DISTINCT FROM` | no | yes | yes | as `<=>`, inverted |

## What is not included

`qbrender.cannot_build()` answers this list at run time. Each of these
is written by hand, with `QbRaw` for the fragment that needs it.

- **Data definition.** `CREATE`, `ALTER`, `DROP` and `GRANT`. A tool
  that builds those wants a different shape: an ordered, named,
  reversible list of changes.
  [migrate](https://novo-lang.org/packages/migrate) is that tool.
- **Window functions.** `OVER (PARTITION BY … ORDER BY … ROWS …)` is a
  grammar of its own, with frames, exclusions and named windows. Use
  `QbRaw` for the whole `SELECT` item.
- **`GROUPING SETS`, `CUBE` and `ROLLUP`.**
- **`VALUES` as a table source, `LATERAL`, `TABLESAMPLE` and table
  functions.**
- **Locking beyond `FOR UPDATE`.** No `FOR SHARE`, `SKIP LOCKED` or
  `NOWAIT`.
- **Vendor hints and comments.** The sql-engine dialect reads `--` as
  two minus signs, so a rendered comment there would not merely be
  ignored, it would fail to parse.
- **Type translation.** See rule 9.
- **Execution.** Nothing here opens a connection.
  `QbRender.text` and `QbRender.params` go to a driver.
- **A device build.** A statement is a tree of lists of strings, and a
  device that talks to a database talks to it over a network, with the
  query built at the other end.

## Related packages

- [postgres-nv](https://novo-lang.org/packages/postgres-nv),
  [mysql-nv](https://novo-lang.org/packages/mysql-nv) and
  [sqlite-nv](https://novo-lang.org/packages/sqlite-nv) run what this
  package renders. Each takes the text and the value list separately,
  which is the shape `QbRender` answers in.
- [sql-engine-nv](https://novo-lang.org/packages/sql-engine-nv) is a
  SQL lexer, parser and executor, and it is the fourth dialect here.
  Its tree is a parser's output and covers every statement it accepts,
  including the data definition this package does not build. This
  package's tree is a builder's input: four statements, with the
  values in it.
- [migrate](https://novo-lang.org/packages/migrate) plans schema
  migrations. It produces the four statements the version table needs
  and nothing else.
- `std.sql` in the standard library opens an SQLite file and runs
  statements against it.

## Tests

```bash
novo test tests/qbbuild_tests.nv     # 15 tests: the builder chain and the refusals
novo test tests/qbdialect_tests.nv   #  9 tests: the table, one fact at a time
novo test tests/qbrender_tests.nv    # 20 tests: the bytes, and the values beside them
```

The reference implementations are SeaQuery for the dialect table, sqlx
for values in the tree, and SQLAlchemy Core for keeping the statement
separate from the connection.

No test opens a database. Every assertion is on bytes and on the value
list. The suite checks that one tree renders `$1` for PostgreSQL and
`?` for SQLite with the values in the same order, that the values come
out in the order their placeholders appear, that a join with no `ON`
is refused, that `IN ()` is refused, that an `INSERT` with no column
list is refused, that a `RETURNING` clause is reported as unsupported
for MySQL rather than dropped, and that an identifier needing quotes
is refused for the sql-engine dialect rather than written bare.

The tests compile today and fail at run, each on the
`not implemented: query-builder-nv.<module>.<fn>` panic that is its
body. That is the expected state of an interface release. They turn
green one at a time as bodies land.

## Implementation status

Nothing is implemented. Every function below is a `todo()`.

| Module | Public surface |
| --- | --- |
| `qbast` | `target_table`, `is_write`, `tables_of`, `binds_of`, `raw_count`, `depth` |
| `qbbuild` | The five value constructors; `eq`, `ne`, `lt`, `gt`, `le`, `ge`, `in_values`, `like`, `escape_like`, `is_null`, `is_not_null`, `between`, `all_of`, `any_of`; `select`, `from`, `from_select`, `column`, `column_as`, `columns`, `distinct`, `where_`, `and_where`, `or_where`, `join`, `group_by`, `and_having`, `order_by`, `order_by_nulls`, `limit`, `offset`, `with_cte`, `union`; `insert`, `values`, `insert_select`, `on_conflict_ignore`, `on_conflict_update`, `insert_returning`; `assign`, `assign_expr`, `update`, `set`, `update_where`, `update_returning`; `delete`, `delete_where`, `delete_returning`; `page`, `count_of` |
| `qbrender` | `options`, `with_pretty`, `with_require_where`, `with_forbid_raw`, `render`, `render_str`, `render_expr`, `check`, `unsupported`, `unguarded`, `render_len`, `write_ident`, `write_literal`, `raw_sites`, `cannot_build` |
| `qbdialect` | `supports`, `name`, `feature_name`, `features`, `dialects`, `placeholder`, `quote_char`, `is_bare_ident`, `is_reserved`, `reserved_words`, `boolean_literal`, `blob_prefix`, `blob_suffix`, `backslash_escapes` |
| `qberror` | `fault`, `kind_name`, `message`, `is_dialect_limit`, `dialect_limits`, `malformed` |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
