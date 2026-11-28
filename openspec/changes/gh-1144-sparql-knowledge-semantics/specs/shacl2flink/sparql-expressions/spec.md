## MODIFIED Requirements

### Requirement: A compiled expression answers what SPARQL answers
A constraint SHALL report an entity exactly when the query, with `$this` bound to it, has a solution, for every supported form and whatever the first pattern of the query is, also for a class that the knowledge does not declare. The offline checks and Flink SHALL agree.

#### Scenario: OPTIONAL first
- **WHEN** a query starts with an `OPTIONAL` and compares what it reads with what a later pattern reads
- **THEN** it answers the same as with the `OPTIONAL` last

#### Scenario: Only a NOT EXISTS
- **WHEN** a query consists of one `FILTER NOT EXISTS`
- **THEN** it finds the entities of the target class for which the pattern has no solution

#### Scenario: The same attribute inside
- **WHEN** a query reads an attribute and, in an `EXISTS` or `NOT EXISTS`, the same attribute with another condition
- **THEN** the inner pattern looks for an instance of its own

#### Scenario: OPTIONAL with a FILTER
- **WHEN** an `OPTIONAL` holds a pattern and a `FILTER` that fails
- **THEN** the entity stays, without what the `OPTIONAL` would give

#### Scenario: OPTIONAL with several patterns
- **WHEN** an `OPTIONAL` reads two attributes of an entity that has one of them
- **THEN** the entity stays, without either

#### Scenario: Conditions
- **WHEN** a query combines conditions with `&&`, `||`, `!`, `BOUND`, `IN` or `NOT IN`
- **THEN** they hold as SPARQL says

#### Scenario: The knowledge
- **WHEN** a query reads the knowledge with `<a>/<b>`, `^<a>` or `rdfs:subClassOf*`
- **THEN** it answers what these paths say

#### Scenario: A class the knowledge does not name
- **WHEN** a query walks `rdfs:subClassOf*` to or from a class that the knowledge does not name as a class, such as a value an attribute holds
- **THEN** the path holds for that class itself, its zero-length step, as for a class the knowledge names

#### Scenario: A target class the knowledge does not declare
- **WHEN** the target class of a constraint is not declared by the knowledge
- **THEN** the entities of that class are validated, and an entity for which the query has a solution is reported

#### Scenario: A rule on a target class the knowledge does not declare
- **WHEN** the target class of a rule is not declared by the knowledge
- **THEN** the rule writes its attributes for the entities of that class for which the query has a solution

#### Scenario: A variable without a value
- **WHEN** a condition compares a variable that an `OPTIONAL` gave no value, also negated
- **THEN** that comparison holds neither way, and `!bound(?v) || ?v > 5` holds for the entity

#### Scenario: Another name
- **WHEN** a query gives a value, or a number it computes, another name with `BIND`
- **THEN** it is compared under that name as what it is

#### Scenario: An IRI from the knowledge
- **WHEN** a query reads an IRI from the knowledge and then asks, also in an `EXISTS` or `OPTIONAL`, for an attribute with that IRI as its value
- **THEN** the attributes with that value are found

#### Scenario: One value in two places
- **WHEN** a query reads one variable from two attributes
- **THEN** there is a solution only where both hold the same value

#### Scenario: A certain entity
- **WHEN** a query names an entity by its IRI and reads an attribute of it
- **THEN** the attribute of that entity is read

#### Scenario: A value a rule writes
- **WHEN** a rule writes a value of an attribute that is an IRI in one shape and a literal in another
- **THEN** it is written as what the value is

#### Scenario: An entity compared
- **WHEN** a query compares an entity with an IRI or with another entity
- **THEN** they are equal exactly when they are the same

#### Scenario: On Flink
- **WHEN** queries of these forms run as constraints on Flink, on entities written through the context broker
- **THEN** the alerts are those of the entities SPARQL finds, also for a target class that the knowledge does not declare
