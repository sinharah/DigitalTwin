Issue: #1144

## Why

A SPARQL constraint SHALL report an entity exactly when the query has a solution for it (`sparql-expressions`). Two forms break that today, both because the compiled SQL reaches a class only through rows of the knowledge (the `rdf` table):
- `rdfs:subClassOf*` misses its zero-length step for a class the knowledge doesn't name as a class;
- a SPARQL shape whose target class the knowledge doesn't declare finds none of its entities.

The second fails silently. Such entities stay "ok" forever, and a rule on such a class never fires, while property shapes on the same class work, because they compute the target's subclasses when the SHACL is compiled.

## What Changes

- **`rdfs:subClassOf*` takes its zero-length step for every class.** A query that walks the path to or from a class the knowledge doesn't name as a class answers what SPARQL answers, as for a declared one. A path with both ends unbound stays refused by name.
- **The target class of a SPARQL constraint or rule is resolved when the SHACL is compiled,** to the class and its subclasses in the knowledge, as property shapes already do. It is no longer a join with the knowledge in the running job.
  - Entities of a target class the knowledge doesn't declare are now validated, and rules on it fire.
  - Each SPARQL constraint and rule has one join with the knowledge fewer.
- **The `ladder` method gets the same,** so that both methods keep reporting the same (`sparql-compilation`).
- **Already done:** point 1 of the issue, a blank node in the object of a knowledge pattern matching only blank nodes, was fixed by gh-1107. This change only records it; there is no code for it.
- **Documentation:** the `subClassOf*` limitation in `docs/supported-features.md` goes.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `shacl2flink/sparql-expressions`: the requirement "A compiled expression answers what SPARQL answers" no longer limits `rdfs:subClassOf*` to classes the knowledge names. It also gets scenarios for a target class the knowledge doesn't declare, for a constraint and for a rule.

## Out of Scope

- Changing how property shapes resolve their targets: they already resolve them when the SHACL is compiled.
- Loading the shapes into the `rdf` table, or any change to what the knowledge table holds. That is the alternative the issue rejected (a1), and the reduction of the knowledge is #1153.
- `rdfs:subClassOf*` with both ends unbound, and other property paths: they stay as they are.
- The streaming cost of re-snapshots of the knowledge (#1152).

## Impact

- **Components:** shacl2flink.
  - The SQL compilation of SPARQL constraints and rules, for the `classic` and `ladder` methods.
  - Its offline SQLite tests.
  - The Flink e2e suite.
  - `docs/supported-features.md`.
- **Contracts between components:** none change. The alerts and the constructed attributes keep their form.
- **Deployments:** a KMS built again after this change compiles different SQL for its SPARQL shapes. `make flink-deploy` sees a changed package (fingerprint, #1165) and replaces the validation job once.
- **Behaviour:** packages with SPARQL shapes on target classes their knowledge doesn't declare start reporting those entities. Violations that went unreported until now raise alerts.
