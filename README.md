# Postgres agent policy conformance vectors

A language-neutral corpus of `(policy, statement) -> expected decision` for wire-level Postgres policy enforcement: the decisions a proxy, an MCP server, or a driver shim has to reach when it is standing between an AI agent and a database.

Apache-2.0. Use them, vendor them, disagree with them in public.

## Why this exists

Every product in this space makes the same claim: your agent only gets what the policy allows. The claim is unfalsifiable as stated. Read-only, table allowlists, and PII masking all sound like one thing each and are actually a few dozen decisions, most of which only show up on constructs nobody demonstrates: a masked column reached through a `UNION` branch, a write hidden in a `WITH` clause, `row_to_json(u)` laundering a whole row past a column-keyed mask, a locking `SELECT ... FOR UPDATE` that is a read by syntax and a write by effect.

A corpus makes the claim checkable. Run it, publish where you differ, and the argument moves from marketing to a diff.

## What is in here

- `vectors/v1.json` : the corpus. 44 cases across 10 profiles.
- `SPEC.md` : what each field means and how to run the corpus.
- `LICENSE` : Apache-2.0.

Cases are grouped by what they exercise:

| Group | Cases | What it pins |
| --- | --- | --- |
| `read_only/` | 9 | Access mode, including the two shapes that look like reads: a data-modifying CTE and a locking select. |
| `statement_allow/` | 2 | That the statement allowlist narrows within the access mode and never widens past it. |
| `fail_closed/` | 5 | Empty, comment-only, truncated, and non-SQL input, plus a batch containing one refused statement. |
| `allowlist/` `denylist/` | 8 | Relation matching: bare against schema-qualified, case, joins, subqueries, and denylist precedence. |
| `mask/` | 10 | Column masking keyed on the source relation, through aliases, stars, set operations, CTEs, whole-row wrapping, and `RETURNING`. |
| `row_filter/` | 4 | Predicate injection, including into a statement that already has a `WHERE` and into every reference in a join. |
| `migration/` | 3 | Lock-taking DDL against its safe equivalent. |
| `max_affected/` | 3 | Refusing a write whose row count cannot be bounded from the statement alone, and permitting one that can. |

The verdict distribution is deliberately mixed (22 block, 9 mask, 3 row-filter, 10 allow). A corpus that only blocks proves nothing about over-blocking, and one that only allows proves nothing about enforcement. A test fails if any of the three groups empties out, and another fails if these four counts stop matching the corpus.

The group a case sits in is not its verdict. `mask/whole_row_to_json` is in the `mask/` group because masking is what it exercises, and its verdict is `block`: an expression wrapped round a masked value is refused rather than nulled, for the reason `SPEC.md` gives under pass 2.

## These are one engine's answers

The vectors are generated from PgBeam's policy engine, which enforces at the PostgreSQL wire protocol and parses with the PostgreSQL parser itself rather than with a regular expression. They are a reference implementation's behaviour, not a standard, and they are published as the former.

Two consequences worth being explicit about.

**Some cases record conservatism rather than a requirement.** See `mask/unprojected_column_still_reported`: the statement does not project the masked column and the engine reports it as masked anyway. Over-reporting is the safe direction. An implementation that returns a plain allow there is not wrong, but it is not identical either, so the case is published rather than hidden.

**Disagreement is the useful output.** If your engine differs on a case, the interesting question is which behaviour is safer, not which one matches this file. Open an issue with the case id.

**A pass is not a guarantee.** `SPEC.md` lists what these vectors cannot decide, and it also names one gap on the near side of that line: statements that move what a mask rule points at, rather than reading a value through it. Read that section before quoting a conformance result at anybody.

## Running them

Nothing here is executable, on purpose: a runner in one language is a runner most people cannot use. `SPEC.md` describes the format in about a page, and a runner is a loop over `cases`.

The shape:

```
for each case:
    profile  = profiles[case.profile]
    decision = your_engine.evaluate(case.sql, profile)
    compare(decision, case.expect)
```

## Versioning

`vectors/v1.json` carries a `version` field. Cases are added freely; a case's expectation changes only alongside a version bump, so a conformance report against v1 stays meaningful.

## Contributing

Issues and pull requests are welcome here. An issue is the right place to start for a bug, a wrong doc, or a missing capability; say what you ran, what happened, what you expected, and which version you were on.

Do not open a public issue for a suspected security vulnerability. Email security@pgbeam.com, or report it privately from this repository's Security tab.

## License

Apache 2.0. See [LICENSE](LICENSE).
