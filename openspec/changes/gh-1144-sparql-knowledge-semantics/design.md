# Design

## Context

See proposal.md (Why) and the delta of `shacl2flink/sparql-expressions`.

How the two methods reach a class today:

- **classic, the target:** the target of a SPARQL constraint or rule is joined in through two triples added to the query's first pattern group: `$this a ?r . ?r rdfs:subClassOf <T>` (`lib/sparql_to_sql.py`).
  - The second triple is a join with the `rdf` table. The knowledge holds `<C> rdfs:subClassOf <C>` only for classes it names as classes (`utils.transitive_closure`). A target it doesn't declare therefore matches nothing.
  - Constraints are also wrapped in a `type` filter with the target and its subclasses, computed from the knowledge at compile time (`shacl_sparql_to_sql.py`, `sparql_get_all_subclasses`). So no entity of an undeclared target ever gets a verdict, and its `ok` row is all that is left.
  - Rules have no such wrapper (`shacl_construct_to_sql.py`) and never fire for such a target.
- **classic, `rdfs:subClassOf*`:** it is compiled as one plain `rdfs:subClassOf` triple, an inner join with `rdf` (`lib/bgp_translation_utils.py`). The knowledge's reflexive closure supplies the zero-length step, but only for classes it names.
- **Property shapes:** they resolve their targets from the knowledge at compile time and don't join `rdf` for it, so they already validate undeclared targets.
- **ladder:**
  - Its constraints already use a compile-time list of types (target plus subclasses, `lib/sparql_ladder.py`).
  - Its rules join the knowledge with `rdfs:subClassOf <T>`, the same gap as classic.
  - It compiles `rdfs:subClassOf*` as the plain predicate, also without a zero-length step.
  - `sparql-compilation` requires both methods to report the same for every query both support.
- **Point 1 of #1144** (a blank node in a knowledge pattern) was fixed by gh-1107 and is tested in `tests/test_sparql_semantics.py` ("a blank node of the knowledge").

## Goals / Non-Goals

**Goals:**
- The target of every SPARQL constraint and rule, of both methods, is a compile-time list of types: the target class and its subclasses in the knowledge, the target itself included whether or not the knowledge declares it. It is no longer a join with `rdf`.
- `rdfs:subClassOf*` with one end bound has its zero-length step for every class, in both methods.
- classic and ladder keep reporting the same; the parity tests stay green with the new cases.

**Non-Goals:**
- A different closure or different contents of the `rdf` table.
- Any change to property shapes.
- A plan for `rdfs:subClassOf*` with both ends unbound (it stays refused).

## Decisions

### D1: The target as a compile-time list of types (option a2 of #1144, decided)

The two added triples become one, `$this a ?r`, which is the entity's own `type` and is needed anyway. `?r` is restricted to the list of types: the target class and `sparql_get_all_subclasses` of it, as the constraint wrapper already computes it. In SQL that is `type IN (...)` on the entity the query starts from. The rule path gets the same restriction, which it lacked.

The knowledge changes only with the KMS, which is compiled with it, so a compile-time list is never stale while the job runs. A redeploy compiles both again (fingerprint, #1165).

Alternatives:
- **(a1) Add `<T> rdfs:subClassOf <T>` to the `rdf` table for every target class.** It needs the shapes when the knowledge table is built, and so changes the Makefile and the helm charts. It also fixes only the target, not `subClassOf*`. Rejected in #1144.
- **(a3) Make the added triple `rdfs:subClassOf*` and use D2's outer join.** One mechanism for both gaps, but an outer join on every SPARQL shape's hot path. Retractions and re-insertions follow whenever `rdf` rows arrive after the entities: at start-up and at every re-snapshot (#1152). Rejected in #1144.
- **(c) Refuse undeclared targets.** It would break packages whose property shapes work today, and only in the SPARQL shapes. Rejected.

### D2: The zero-length step of `rdfs:subClassOf*` as an outer join with a fallback

`?s rdfs:subClassOf* <O>`, with `?s` bound by an earlier pattern and `<O>` a constant or bound:
- `rdf` is joined LEFT on `subject = ?s AND predicate = rdfs:subClassOf AND object = <O>`;
- the pattern holds where `COALESCE(rdf.object, ?s) = <O>`.

With `<O>` unbound, the pattern binds `<O>` to `COALESCE(rdf.object, ?s)`; the mirror form, with the object bound and the subject unbound, uses `COALESCE(rdf.subject, <O>)`.

This is exact. The closure gives every class that appears in an `rdfs:subClassOf` triple its reflexive row, so a class without any `rdfs:subClassOf` row reaches only itself, and that is the zero-length step the fallback supplies.

A LEFT JOIN cannot come first in the FROM list, so the pattern is placed after the table that binds its bound end. Both ends unbound is refused by name, as now: in SPARQL it would match every node of the graph.

Alternatives:
- **Reflexive rows in `rdf` for every constant at an end of a `subClassOf*` pattern in the shapes.** Like (a1), it needs the shapes in the knowledge table. It also misses a class that only data delivers (an attribute's value).
- **A UNION of the step and the path.** It doubles the statement, and the union's two branches would retract and insert against each other on Flink.

### D3: ladder mirrors D1 and D2 in this change

- **Rules:** they get the compile-time type list, as ladder's constraints already have, instead of the `rdfs:subClassOf <T>` leaf.
- **`rdfs:subClassOf*`:** it gets its zero-length step with the same outer-join-with-fallback semantics, in ladder's plan form.

Kept in this change because `sparql-compilation` ("The methods report the same") would otherwise be broken by the new test cases: the parity tests compare both methods on the SQL test models. ladder is alpha; its owners review this part.

Alternative: **a follow-up issue for ladder,** with the new cases excluded from the parity tests until then. Rejected, because it leaves a spec requirement knowingly broken on `main`.

## Risks / Trade-offs

- **[Behaviour change] Packages with SPARQL shapes on undeclared targets start raising alerts** for violations that went unreported. → That is the fix; the PR names it, and the alerts are correct ones.
- **[Outer join churn] D2's outer join can retract and re-insert when `rdf` rows arrive after the entities.** → It is limited to queries that say `rdfs:subClassOf*`; the target no longer joins `rdf` at all (D1). The Flink e2e covers a `subClassOf*` query across a re-snapshot.
- **[Plan order] A LEFT JOIN placed before its bound end would be invalid SQL.** → The translation orders it after the binding table. An offline test covers a query whose first pattern is the `subClassOf*` one.
- **[Fewer joins changes the plan] Each SPARQL shape loses one `rdf` join,** so the deployed job differs. → Expected; the fingerprint replaces the job once at the next deploy (#1165). The e2e asserts the plan has one join fewer per SPARQL shape.
- **[ladder] The ladder's plan form differs from classic's SQL.** → The parity tests on the new models hold both to the same alerts and constructed attributes.

## Migration Plan

Nothing to migrate. A KMS built again compiles the new SQL; `make flink-deploy` replaces the validation job once and resyncs the alerts, as for any changed package. Rollback is the previous build of the KMS.

## Conflicts with existing specs and documentation

- `sparql-expressions`, scenario "The knowledge": its "for every class that the knowledge names" limitation goes, in the delta.
- `docs/supported-features.md` states the same limitation and is updated.
- `sparql-compilation` is not changed; D3 keeps it true.
