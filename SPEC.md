# Vector format

`vectors/v1.json` is one JSON document.

```json
{
  "version": 1,
  "about": "...",
  "profiles": [ ... ],
  "cases": [ ... ]
}
```

## Profiles

A profile is a policy as an operator writes it, not as an engine compiles it.

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | Referenced by a case's `profile`. |
| `description` | string | Prose, for humans reading a failure. |
| `access_mode` | `read_only` \| `read_write` | Whether writes and DDL are permitted at all. |
| `statement_allow` | string[] | When non-empty, restricts allowed statement kinds. It only ever narrows: a write listed here is still refused under `read_only`. |
| `statement_deny` | string[] | Blocks statement kinds. Takes precedence over `statement_allow`. |
| `table_allowlist` | string[] | When non-empty, only these relations may be referenced. **An absent allowlist means no restriction**, so the restriction is the presence of the list, never its emptiness. |
| `table_denylist` | string[] | Relations that may never be referenced. Takes precedence over the allowlist. |
| `masking_rules` | object[] | `{table, column, kind}` with `kind` one of `redact`, `null`, `hash`. Keyed on the **source** relation and column, never on the output name. |
| `row_filters` | object[] | `{table, predicate}`. The predicate is ANDed into every reference to the relation. |
| `max_affected_rows` | int | Refuses writes whose row count cannot be bounded from the statement alone (a whereless write, a data-modifying CTE). It is **not** a static row estimate: a bounded write is permitted here whatever number the value carries, and the count itself is applied at runtime, outside what these vectors can express. Absent means the check does not run. |
| `migration_safety` | `warn` \| `block` | Whether DDL flagged as lock-taking is refused or merely noted. |

Relation names are matched case-insensitively, and bare (`users`) and schema-qualified (`public.users`) spellings of the same relation both match.

Omitted fields carry the "absent" meaning in the table above, which is not always the same as an empty value. That distinction is the single most common source of a wrong answer: an absent allowlist permits everything, while an empty allowlist, if an implementation represents one, must not.

## Cases

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | `group/name`. Stable, so a report can cite it. |
| `profile` | string | A profile `id`. |
| `note` | string | Present when the case is not self-evident. Read these. |
| `sql` | string | The statement, exactly as an agent would send it. May be empty, malformed, or a multi-statement batch. |
| `expect` | object | Below. |

## Expectations

| Field | Type | Meaning |
| --- | --- | --- |
| `verdict` | `allow` \| `block` \| `mask` \| `row-filter` | The headline decision. `mask` and `row-filter` both permit the statement while changing what it returns. |
| `rule` | string | Machine-readable tag for the deciding rule (`ok`, `read_only`, `destructive_ddl`, `table_not_allowed`, and so on). Compare it when your engine has an equivalent; treat a mismatch here as weaker evidence than a `verdict` mismatch. |
| `masked_columns` | string[] | `name:kind`, sorted. See the note below: this is a union of two kinds of entry, not a list of output column names. |
| `row_filtered` | bool | Whether a predicate was injected. |

### How `masked_columns` is built, and why it is wider than the output

The name to the left of the colon is not always a column the client sees. The list is produced by two passes and holds the union of both.

Neither pass runs on a statement that was refused: a `block` carries an empty `masked_columns`, because nothing is masked when nothing is evaluated. `mask/whole_row_to_json` reports `[]` for that reason.

**Pass 1, the floor.** Every masked column of every relation the statement touches is listed under its own name, carrying the rule's `kind`, whether or not the column reaches the output at all. This is why `mask/unprojected_column_still_reported` reports `email:redact` for `SELECT id FROM users`: `users` is touched, so its masked column is named.

**Pass 2, provenance.** Each output column whose expression references a masked source column is also listed, under its **output** name:

- a bare reference, aliased or not, keeps the rule's kind, so `SELECT email AS contact FROM users` adds `contact:redact`.
- a projection that wraps nothing but still cannot be shown to be value-preserving fails closed: the whole output column is nulled, giving kind `null` regardless of what the rule says. `mask/scalar_subquery` is the case, `SELECT (SELECT email FROM users LIMIT 1) AS e`, which adds `e:null`. A bare whole-row reference is nulled the same way, under the relation's own alias: `SELECT u FROM users u` adds `u:null`. Wrapping that same reference in anything moves it to the bullet below, so `SELECT u::text FROM users u` is blocked. Neither spelling has a vector of its own beyond `mask/scalar_subquery`.
- an expression wrapped **round** a masked value does not reach pass 2 at all, because the statement is refused: verdict `block`, rule `masked_column_expression`. `SELECT row_to_json(u) FROM users u` is the published case (`mask/whole_row_to_json`), and `SELECT email || '' FROM users` is the same shape. Nulling the cell is not sufficient for these: the expression is still evaluated on the raw value, so whether it errors answers a question the agent wrote into the predicate, one bit per query, and no mask can redact that.
- a **VALUES list** is refused under the same rule, and there for a bare reference too: `VALUES ((SELECT email FROM users WHERE id = 1))` is `mask/values_list`. A VALUES list is a projection, but PostgreSQL names its outputs `column1`, `column2` and so on, so neither pass has a name to attach a mask to and the raw value would reach the client. Every nesting is the same parse node, so the rule covers a standalone `VALUES`, one in a FROM subquery, and one defining a CTE.

- an **alias column list** on a relation reference is refused under rule `masked_column_rename`: `SELECT x FROM users AS u(x)` is `mask/alias_column_list`. The list renames the relation's columns by POSITION, so the masked value reaches the client under a name neither pass can attach a rule to, and the statement names no masked column for either pass to find. An engine that reads SQL without a catalog cannot tell which alias covers which source column, so masking the right one is not available to it and refusing is the only sound answer. Scoped to a relation reference, which is the one FROM item whose column names come from the catalog: a subquery or CTE carries its own target list, so pass 2 follows a rename through either and `SELECT x FROM (SELECT email FROM users) AS t(x)` stays a `mask`. An alias that renames the relation alone moves no column name and is unaffected, which is `mask/relation_alias_still_masked`.

Two consequences to build to. A profile whose only rule is `kind: redact` still produces `:null` entries, because `null` here is the fail-closed outcome of pass 2 rather than a mask kind anyone configured. And a single output column can contribute two entries: `mask/aliased_column` reports `["contact:redact", "email:redact"]`, the alias from pass 2 and the source name from pass 1.

An implementation that reports only the client-visible names is not wrong about what the client receives. It will differ from these vectors, and that is a reporting difference rather than a data one, which is why `verdict` is the field to compare first.

## Running

```
for each case:
    profile  = profiles[case.profile]
    decision = your_engine.evaluate(case.sql, profile)
    compare(decision, case.expect)
```

Compare `verdict` first. It is the field that decides whether data left the database. `masked_columns` is next. `rule` is a diagnostic: a different engine may reach the same verdict by a differently named rule, and that is not a failure worth reporting on its own.

## What the vectors do not cover

Everything whose answer depends on state a single statement cannot carry:

- per-credential query and egress budgets
- human-in-the-loop approvals
- write-mode routing (rollback and sandbox branches)
- the affected-row cap's own runtime count, as opposed to the statements it refuses because it could never count them
- honeytokens and other project-level overlays

Those are real enforcement and they are not decidable from `(policy, statement)`, which is the boundary this corpus draws. A conformance pass says nothing about them.

### One gap inside the boundary

The line above is about what a single statement cannot carry. There is also a gap on the near side of it, decidable from `(policy, statement)` and still not covered here, and it is worth naming because it sits directly under the masking cases.

Mask rules are keyed on schema, table and column. A statement that changes what those names point at, rather than reading a value through them, attacks the rule instead of the data. The two loudest shapes are refused: renaming a masked column or its table, and naming a masked column in a constraint or an index definition, where the server would evaluate the definition against every stored row and hand back either one bit about the real values or a raw copy of them.

Three shapes in the same family are still permitted by this engine today and have no vector:

- a bare narrowing type change, which is validated against every stored row and fails if any is too long, so repeating it binary-searches a length. The spelling that carries a `USING` expression is already refused, because that expression reads the masked column; only the bare form survives.
- attaching a partition, which brings rows under a relation name the rules were written against
- a function body, which arrives as a string this engine does not parse, so no rule inside it is evaluated at all

An implementation reading this corpus should not conclude that a masking pass means masked values cannot be reached by schema changes. It means the statements listed here reach the listed verdicts.
