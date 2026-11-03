# Changelog

All notable changes to query-builder-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-13

The implementation.  Every `pub fn` has a body, `stability` is
`experimental`, and the three suites the interface release shipped pass
— stamped, and rewritten `redundant-bool` clean, both recorded under
Changed below and neither an assertion.
`tests/query_builder_nv_edge_tests.nv` is added with them:
it asserts the boundary the README claims — a hostile value travels as
a value and cannot change the statement's shape.

### Added

- **The render**, `qbrender.render`, `render_str`, `render_expr` and
  `render_len`: one walk writes the bytes and collects the parameters,
  so the text and the list cannot get out of step.  `check` is the same
  walk keeping every fault where `render` stops at the first, and
  `unsupported` filters those faults down to the dialect's own limits.
- **The writer pair**, `write_ident` and `write_literal`, published
  because they are the pieces a caller writing their own SQL most often
  needs: `write_ident` quotes a name that needs quoting, refuses one
  holding the dialect's own quote character or a zero byte rather than
  doubling, and refuses one that needs quoting where the dialect has no
  quote character at all; `write_literal` doubles a quote in all four
  dialects and doubles a backslash only in MySQL.
- **The walk in `qbast`** — `tables_of`, `binds_of`, `raw_count`,
  `depth` — over one traversal written once and shared with the
  render's order, so `binds_of` is the parameter list before it is
  written and the four answers cannot drift apart.
- **The builder chain completed**: `page` and `count_of`, the four
  statement writers, the CTEs, the set operations and the conflict
  clauses.
- **The boundary suite**: 36 tests.  Its first two render the same
  statement with a hostile value and with a harmless one and assert the
  two are byte-identical with the value found whole in the parameter
  list, so they fail if the renderer ever puts a value into the text.

### Changed

- **The three suites the interface release shipped are stamped** — the
  header that told a reader the file fails on purpose is deleted, and
  nothing else about them changed but the next item.
- **Their twenty-six `x == false` predicates are rewritten `not x`.**
  That is `redundant-bool`, the one rule `novo.lint` fires on the old
  suites, and it is the linter's own suggestion, so no predicate
  changed.  Every site was confirmed pre-existing against
  `HEAD:tests/` first: one in `qbbuild_tests.nv`, twenty-one in
  `qbdialect_tests.nv`, four in `qbrender_tests.nv`.  `grep -rn
  '== false' tests/ src/` answers zero here now, which is the whole set
  and not only the sites the linter reports.

### Known

These are measured, not guessed; each is a property of this release
that a caller should know before depending on it.

- **A rendered buffer ends with a zero byte.**  `render` and
  `render_len` count it; `render_str` drops it.  A caller handing
  `QbRender.text` straight to a driver has to slice it off.
- **A condition passed for a `CROSS JOIN` is dropped.**  `CROSS JOIN t
  ON …` is a syntax error in every dialect here, and `qbbuild.join` has
  no spelling for "no condition", so the drop is silent.
- **`count_of` keeps the statement's set arms**, so counting a
  `UNION` counts each arm rather than the union.  Its columns, ordering,
  limit and offset are dropped as documented.
- **`render_expr`'s `first_param` is the number of parameters already
  written, not the number the first one of this expression takes**:
  passing 6 writes `$7`.  The doc comment reads one off; the code and
  the suite agree with each other and not with the sentence.
- **`DISTINCT ON (…)` is written as one clause**, so a select carrying
  both `distinct` and `distinct_on` gets one `DISTINCT` and not two.
- **`INSERT … SELECT` writes its source parenthesized**, `INSERT INTO t
  (a) (SELECT …)`, because the source goes through the same writer the
  `FROM` clause uses.  Every dialect here accepts it and it is not the
  canonical spelling.
- **`qbbuild.page` does not check its arithmetic.**  A negative limit or
  offset is written as absent, because `-1` is how the struct spells
  "none", so `page(q, -1, 25)` renders `LIMIT 25` with no `OFFSET` —
  page 0's rows, silently — and `page(q, 1, -25)` renders no `LIMIT` at
  all, which is the whole table.  A page number and a size come from a
  request in the case this shorthand exists for, and nothing here
  refuses one.
- **`qbrender.write_ident` refuses, for the sql-engine dialect, any
  bare identifier reserved in ANY of the four dialects** — the union of
  the reserved-word lists, not sql-engine's own.  A word reserved only
  in MySQL is refused there too.  The refusal errs toward the safe side
  and is stricter than that lexer needs to be.
- **Two of the eighteen feature rows are never reported.**  Nothing in
  this package writes a comment, and an identifier the sql-engine
  dialect cannot quote is reported as `QbUnquotableIdent`, not as
  `QbUnsupported(QbFeatQuotedIdent, …)` or `…QbFeatAnonymousSubquery…`
  — sql-engine reports `QbSubqueryNeedsAlias` for the second.  So
  `unsupported` cannot name those rows.  The seven unreachable arms in
  `feature_refuses` carry `cov: skip` markers saying so.
- **MySQL's rewrite of `||` as `CONCAT(a, b)` is not reported**, because
  there is no feature row for a concatenation operator: a port reading
  `unsupported` sees the `IS DISTINCT FROM` → `<=>` rewrite and not
  this one.
- **`write_literal` cannot refuse.**  No value literal records a
  refusal — the one feature writing one can use, the boolean literal,
  is reported and not refused — so its `Err` position is unreachable
  and is marked `cov: skip` in the source.
- **Two workarounds for open compiler defects are in the source**, each
  citing its filing.  `qbrender.unguarded` asks its question through
  the helper `absent` because a `match` nested inside a `match` arm
  strands a cell
  (`memory-perceus/a-nested-match-with-a-method-answer-arm-leaks-the-enclosing-subject`);
  and the boundary suite binds a list element to a `let` before reading
  a `Str` field off it, because the same read through a temporary
  answers a freed borrow
  (`memory-perceus/a-str-field-read-off-a-temporary-lists-element-answers-a-freed-borrow`,
  filed from this package).

This release requires novo 0.23.0 or later.

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
