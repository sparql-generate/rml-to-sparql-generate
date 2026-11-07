# rml-to-sparql-generate

An RML-to-SPARQL-Generate translator **written in SPARQL-Generate**: a
`TEMPLATE` query that reads an RML mapping as its default graph and writes the
text of an equivalent SPARQL-Generate `GENERATE` query.

```sh
mkdir -p work/dataset && cp mapping.ttl work/dataset/default.ttl
cp -r rml2sparql-generate.rqg templates iterators work/
(cd work && java -jar sparql-generate.jar -d . -q rml2sparql-generate.rqg \
      -b http://example.com/base/ -o generated.rqg)
# then, from the directory of the data, with the same base:
java -jar sparql-generate.jar -d data -q generated.rqg -b http://example.com/base/ -o out.nt -fo NT
```

Run the translator from its own directory: its named templates
(`templates/*.rqg`, `iterators/*.rqg`) are resolved against the current
directory. `-b` gives the base IRI of the mapping: it resolves the relative
IRIs that the mapping generates, and it names the sources of the generated
query (`ITERATOR iter:CSV(<student.csv>, ...)`), which the command line maps to
the files of the data directory.

## What it translates

The translation is τ of the formal semantics of SPARQL-Generate
(`rml-translation.md`, companion of the SPARQL-Generate semantics): one
sub-query per triples map, iterating once over its logical source, with one
nested sub-query per class set and per (predicate-object map, predicate map,
object map).

| RML | Generated SPARQL-Generate |
|---|---|
| RML 1.x (`rr:`, `rml:`, `ql:`) and RML-Core (`http://w3id.org/rml/`) | both read |
| logical source, `ql:CSV` / `rml:CSV` | `ITERATOR iter:CSV(<file>, "col", ...) AS ?c_...` (one solution per row) |
| `ql:JSONPath` / `rml:JSONPath` | `ITERATOR iter:JSONPath(<file>, "<iterator>") AS ?it`, then one `ITERATOR iter:JSONPath(?it, "<ref>", "$")` per reference: a multi-valued reference gives one term per value |
| `ql:XPath` / `rml:XPath` | `ITERATOR iter:XPath(<file>, "<iterator>") AS ?it`, then one `ITERATOR iter:XPath(?it, "/*/<ref>/text()")` per reference |
| `rr:constant`, `rml:reference`, `rr:template` (escapes `\{ \} \\`) | constants, variables, `CONCAT(...)` |
| term types `IRI` (IRI-safe encoding, exact on non-ASCII), `URI`, `UnsafeIRI`, `Literal`, `BlankNode` | `IRI(...)` with base-IRI concatenation, `STRLANG`, `STRDT`, `fun:bnode(...)` |
| `rr:language`, `rr:datatype`, `rml:languageMap`, `rml:datatypeMap` | `STRLANG`, `STRDT` |
| `rr:class`, shortcuts `rr:subject`/`rr:predicate`/`rr:object`/`rr:graph` | yes |
| referencing object map, same logical source, no join | the parent's subject, on the same iteration |
| referencing object map with `rr:joinCondition` (`rml:childMap`/`rml:parentMap` too) | the parent is generated once as a graph (`FROM GENERATE`), the join is a basic graph pattern over it; for a CSV child whose subject is a blank node without expression, the join is a sub-query in the child's iteration (the blank node of a row must be the same in both) |
| `rml:baseIRI` (per triples map) | yes |
| graph maps | GENERATE writes triples: only the triples of the **default graph** are generated |
| invalid mappings | `#ERROR: ...` lines at the top of the output (no subject map, two subject maps, literal subject, invalid language tag, invalid template, unbalanced JSONPath, ...) |

Blank nodes. RML gives one blank node per value, across iterations and
triples maps; a SPARQL `BNODE(str)` is fresh for each solution. The generated
query uses `fun:bnode(str)` (SPARQL-Generate 2.2): the same blank node for the
same string during one execution, sub-queries included.

## Optimised translation τ'

The generated query is τ, the proved translation of the formal semantics, followed by
mechanical rewritings; each is exact (the same output up to isomorphism), and is checked
by the conformance suites (no case changes status) and by the benchmark workloads (the
same distinct triples):

| | rewriting | when |
|---|---|---|
| R1 | the sub-queries of an iteration (one for the classes, one per predicate map and object map) are flattened into the WHERE clause of the triples map: each term is bound once per row, and the triples map has one template | CSV sources: a reference has one value per row, so each sub-query has at most one solution |
| R2 | `IRI(?l)` instead of a run-time test for a relative IRI | templates whose constant text starts with a scheme |
| R3 | the triples maps of a CSV source read by several of them share one iteration of the file (their variables renamed apart) | CSV triples maps with a reference, and without a blank node map without expression |
| R3 | the triples maps of one CSV file share one iteration of it: one GENERATE with all their triples, `iter:CSV` over the union of their columns | the same file and reference formulation, each triples map with at least one reference or template, and no blank node map without expression |
| R4 | CSV files are read in batches of 10 000 rows (a streamed output is written batch by batch) | always: the generated queries have no solution modifier |

The translation needs SPARQL-Generate 2.2.3 or later.

## Conformance

Measured by kgc-bench on SPARQL-Generate 2.2.4 (release) and pysparql-generate, which give
the same status on every case, by RDF dataset isomorphism with the expected output of each
case (an expected error passes when an error is raised, with `--fail-on-missing`):

| suite | cases | PASS | N/A (graph maps) | FAIL |
|---|---|---|---|---|
| RMLio/rml-test-cases@803dd3ec, CSV | 39 | 32 | 6 | 1 |
| RMLio/rml-test-cases@803dd3ec, JSON | 40 | 33 | 6 | 1 |
| RMLio/rml-test-cases@803dd3ec, XML | 38 | 31 | 6 | 1 |
| kg-construct/rml-core@82ab28d4, JSON | 76 | 66 | 7 | 3 |

Graph maps need quad templates (`GENERATE { GRAPH ?g { ... } }`), planned for
SPARQL-Generate 2.3; the default graph of those cases is checked. The failures are
errors that only the data reveal (an unknown column or file, a reference to an array, an
invalid IRI), which the generated query does not raise, and `rml:UnsafeIRI` values with
spaces, which the engine rejects as IRIs (Jena's `IRI()`, also behind `<...{?x}>`). A
missing file is an error with `--fail-on-missing` (2.2.3).

## History

The first version of this translator, also a SPARQL-Generate `TEMPLATE`,
dates from 2019 (commit `166c901`). It chained the iterations of all the
triples maps in the top-level query (a Cartesian product of all the logical
sources), joined with `IF` over that product, evaluated references with the
first value only, and relied on the STTL function `st:turtle`, which the
SPARQL-Generate 2 library does not define. The present version keeps its
decomposition into templates (subject map, predicate-object map, term map,
template value, reference formulation) and its principle — the translator is
a SPARQL-Generate query — and replaces the rest.

## Licence

Apache-2.0.
