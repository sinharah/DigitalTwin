# Tasks

## 1. Tests first (classic and ladder, failing on main)

- [ ] 1.1 In `tests/test_sparql_semantics.py`, add the cases of the delta's new scenarios, each compared with rdflib on the same data:
  - `?s rdfs:subClassOf* <C>` from an attribute's value, to a class the knowledge does not name, and to one it names only as an object;
  - the mirror form with the object bound;
  - a constraint whose target class the knowledge does not declare, for which the query has a solution.

  Verified: the new cases fail on `main` and name the disagreement.
- [ ] 1.2 Add a rule case: an `sh:construct` rule on a target class the knowledge does not declare. Verified: on `main` it writes nothing, and the expected attributes are stated.
- [ ] 1.3 Add an offline test that counts the joins with the `rdf` table in the compiled statement of a SPARQL constraint and of a rule, so that the count is pinned for D1. Verified: on `main` the count is one higher than what the test asserts.
- [ ] 1.4 Add a model with an undeclared target class and a `subClassOf*` query to the SQL tests, so that the parity tests compare classic and ladder on it. Verified: `test_sparql_ladder_parity.py` covers the new model.

## 2. classic: the target as a list of types (D1)

- [ ] 2.1 Replace the added triples `$this a ?r . ?r rdfs:subClassOf <T>` by `$this a ?r` restricted to the target class and its subclasses in the knowledge, computed at compile time as the constraint wrapper computes it. Verified: tasks 1.1 (target), 1.3 and the existing SPARQL semantics tests pass.
- [ ] 2.2 Give rules the same restriction (they had no outer type filter). Verified: task 1.2 passes.

## 3. classic: the zero-length step of `rdfs:subClassOf*` (D2)

- [ ] 3.1 Compile `rdfs:subClassOf*` with one end bound as a LEFT JOIN with `rdf` with the `COALESCE` fallback, in both directions, placed after the table that binds its bound end. Verified: the `subClassOf*` cases of task 1.1 pass, including one whose first pattern is the path.
- [ ] 3.2 Keep both ends unbound refused by name. Verified: an existing or added refusal test passes.

## 4. ladder mirrors (D3)

- [ ] 4.1 Give ladder's rules the compile-time list of types instead of the `rdfs:subClassOf <T>` leaf. Verified: `test_sparql_ladder_semantics.py` with the rule case of task 1.2.
- [ ] 4.2 Give ladder's `rdfs:subClassOf*` its zero-length step, with the same semantics as D2. Verified: `test_sparql_ladder_semantics.py` with the `subClassOf*` cases.
- [ ] 4.3 Run the parity tests and the ladder's join count test with the new model of task 1.4. Verified: `test_sparql_ladder_parity.py` and `test_sparql_ladder_joins.py` pass, the join count updated where D1 removes a join.

## 5. Flink end to end

- [ ] 5.1 Add to the e2e KMS a SPARQL constraint on a target class that its knowledge does not declare. In `test-shacl-flink-e2e`, assert that an entity of that class which violates the constraint raises its alert, and that a valid one gets `ok`. Verified: the suite passes on Flink, and its new test fails with `main`'s build.
- [ ] 5.2 Add a `subClassOf*` query to a class the knowledge does not name to the e2e KMS. Assert its alert, also after a re-snapshot of the knowledge (`make shacl-resync`). Verified: the suite passes on Flink.
- [ ] 5.3 Assert in the e2e that the deployed validation job's plan has one join fewer per SPARQL shape than the count recorded from `main`'s build of the same KMS. Read it from the Flink REST plan. Verified: the assertion passes, and the recorded count with `main`'s count is in the test.
- [ ] 5.4 Run the `flink-ladder` shard with the new cases. Verified: green in the PR's `e2e-all` run.

## 6. Documentation

- [ ] 6.1 In `docs/supported-features.md`, drop the limitation of `rdfs:subClassOf*` to classes the knowledge names, and state that undeclared target classes are validated. Verified: the documentation test of `sparql-expressions` ("The supported forms are documented") passes.
- [ ] 6.2 State in the PR that point 1 of #1144 was fixed by gh-1107, and that packages built again replace their validation job once. Verified: in the PR description.

## Workflow follow-up

- Archive the change after the implementation PR is merged (`/opsx:archive`), as a separate PR.
