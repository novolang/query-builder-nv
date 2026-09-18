# Changelog

All notable changes to query-builder-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md).

`QbFault` now declares the `impl Error` its own `Result` positions
require.  `Result<T, E>` has carried the bound `E: Error` since SPEC
§ 3.4, and the compiler enforced it only when `E` was declared in the
module that named it — so `Result<_, qberror.QbFault>` was accepted
across modules with no impl anywhere.  The impl is the signature this
package always meant; nothing else about the interface changed.

Fixed, in the same edit: `qbast.QbCteRef` declared its `name` field as
`String`, which is not a type this language has — the two sibling
variants of `QbSource` both spell it `Str`.  Every build of this
package failed on it, so nothing that depended on 0.0.1 ever compiled.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `qbast` — the whole vocabulary in one module because it is cyclic:
  `QbValue`, `QbExpr` with twenty-four variants, `QbSelect`,
  `QbInsert`, `QbUpdate`, `QbDelete` and `QbStmt`; plus the questions
  worth asking a statement — `target_table`, `is_write`, `tables_of`,
  `binds_of`, `raw_count`, `depth`.
- `qbbuild` — the builder chain: `select`, `from`, `and_where`, `join`,
  `group_by`, `order_by`, `limit`, `with_cte`, `union`; the four
  writers; and the comparisons that bind by construction.
- `qbdialect` — four dialects and eighteen features as one table, plus
  the placeholder, the quote character, the reserved-word lists and the
  backslash-escape rule.
- `qbrender` — `render` into a caller's buffer with the parameters
  beside the bytes, `check` for every fault rather than the first,
  `unsupported` for the port question on its own, and `cannot_build` as
  a list.
- `qberror` — `QbFault` with a path into the statement, and the split
  between a malformed statement and a dialect that cannot say it.

### Known

- **The value goes in the tree, not in a list beside it.** There is no
  API here that takes a parameter list; the list comes out of the
  render, in placeholder order.
- **A dialect is a table, not a branch.** `qbdialect.supports` is total
  over both enums, so adding a dialect is a column and adding a feature
  is a row.
- **sql-engine-nv cannot quote an identifier at all** — its lexer
  recognises neither `"` nor a backtick — so a column named after a
  reserved word is refused there rather than mis-rendered.
- **sql-engine-nv has no placeholder token**, although pager-nv's
  `driver.execute` takes a params list. `supports(QbSqlEngine,
  QbFeatParameters)` answers false and the README states the gap.
- **A join with no `ON` is refused**, `IN ()` is refused, and an
  `INSERT` with no column list is refused — each because the plausible
  repair would silently change which rows the caller gets.
- **An `UPDATE` or `DELETE` with no `WHERE` is NOT refused**, because
  it is a real statement; `unguarded` is the question and
  `require_where` is the switch.
- **`QbRaw` is countable and forbiddable**, because it is the whole of
  the package's attack surface.
- **No DDL, no window functions, no `GROUPING SETS`, no `VALUES` as a
  source, no locking beyond `FOR UPDATE`, no comments**, and
  `cannot_build()` answers the list at run time.
- **No execution, and no dependencies.** The plan's row says "std.sql
  executes it", and that is the right direction: a consumer takes both
  and this package does not know the other exists.
- **No device claim**: a statement is a tree of lists of strings.
