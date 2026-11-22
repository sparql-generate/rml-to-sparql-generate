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
java -jar sparql-generate.jar -d data -q generated.rqg -b http://example.com/base/ -o out.nq -fo NQUADS
```

Run the translator from its own directory: its named templates
(`templates/*.rqg`, `iterators/*.rqg`) are resolved against the current
directory. The generated query writes an RDF dataset: the default graph, and the named
graphs of the graph maps of the mapping (`GRAPH` blocks), hence N-Quads or TriG
(`-fo NQUADS`, `-fo TRIG`); `-fo NT` suits a mapping without graph maps.

`-b` gives the base IRI of the mapping: it resolves the relative
IRIs that the mapping generates, and it names the sources of the generated
query (`ITERATOR iter:CSV(<student.csv>, ...)`), which the command line maps to
the files of the data directory.

## What it translates

The direct translation generates one sub-query per triples map, iterating once over its logical source, with one
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
| graph maps: `rr:graphMap`, `rr:graph`, `rml:graphMap`, `rml:graph` (RML-Core), on the subject map (every triple of the triples map, `rr:class` included) and on each predicate-object map; several graph maps: the triple in each; `rr:defaultGraph`: the default graph | the default graph outside `GRAPH` blocks, `GRAPH <g> { ... }` for a constant, `GRAPH ?g { ... }` for a template or a reference (term type IRI), bound once per row (CSV) or in a sub-query of the predicate-object map, one quad per value (JSONPath, XPath); with a join, the graph term is carried from the child's iteration |
| gather maps (RML-CC, `http://w3id.org/rml/`): `rml:gather`, `rml:gatherAs` (`rdf:List`, `rdf:Bag`, `rdf:Seq`, `rdf:Alt`), `rml:strategy` (`rml:append`, `rml:cartesianProduct`), `rml:allowEmptyListAndContainer`, in a subject map or an object map, nested gather maps, named collections (`rml:template`, `rml:reference`, `rml:constant`, IRI or blank node), members that are referencing object maps, with graph maps | one sub-query per gather map at the top level: a `FROM GENERATE` iterates over the logical source and records each element of each collection with an order key (position of the iteration, member, positions of the values), then the cells (`rdf:first`, `rdf:rest`, `rdf:_n`) are built from the rank of each element in its collection (see below) |
| function-valued term maps (RML-FNML): `rml:functionExecution`, `rml:function` or `rml:functionMap`, `rml:input` with `rml:parameter` or `rml:parameterMap`, `rml:inputValue` or `rml:inputValueMap` (reference, template, constant, function: composition), `rml:return` or `rml:returnMap`; in any term map, language and datatype maps included; `rml:condition` | the SPARQL expression of the function (see below), in place of the lexical form of the term map; a condition gives the term only when it is true |
| a triples map without logical source, of constant term maps (RML-FNML) | a sub-query without iterator, evaluated once |
| logical views (RML-LV): `rml:LogicalView`, `rml:viewOn` (a logical source or another view), `rml:field` (`rml:ExpressionField` with `rml:reference`, `rml:template` or `rml:constant`; `rml:IterableField` with `rml:iterator`; nested fields, `rml:referenceFormulation` to change formulation: JSON in CSV, CSV in JSON), the index keys `#` and `<field>.#`, `rml:leftJoin` and `rml:innerJoin` (`rml:parentLogicalView`, `rml:joinCondition` with `rml:child`/`rml:parent` or `rml:childMap`/`rml:parentMap`, joined fields) | the iteration of the view as binding clauses: one iterator per field, nested over the records of its parent (one solution per logical view iteration), each key bound to a variable that the term maps read as a CSV column; a join is a `FROM GENERATE` of the parent view's iterations, matched in the `WHERE` clause (`OPTIONAL` for a left join) |
| RML-star: `rml:quotedTriplesMap` in a subject map or an object map (star maps), on the same logical source or with `rml:joinCondition`, nested quoted triples, `rml:AssertedTriplesMap`, `rml:NonAssertedTriplesMap` | the triple term `afn:triple(s, p, o)` (Jena, RDF 1.2) of the terms of the quoted triples map on the same iteration; with a join, a `FROM GENERATE` of the quoted triples of the other source, matched in the `WHERE` clause; a non-asserted triples map generates no triple of its own |
| RML-IO sources: `rml:source` as a literal path, `rml:FilePath` and `rml:RelativePathSource` (`rml:path`, `rml:root`); `rml:null` | the file, relative to the base IRI; a value among the null values of the source is no value (CSV) |
| invalid mappings | `#ERROR: ...` lines at the top of the output (no subject map, two subject maps, literal subject, a predicate map or graph map of term type Literal or BlankNode, a literal graph name, invalid language tag, invalid template, unbalanced JSONPath, ...) |

Blank nodes. RML gives one blank node per value, across iterations and
triples maps; a SPARQL `BNODE(str)` is fresh for each solution. The generated
query uses `fun:bnode(str)` (SPARQL-Generate 2.2): the same blank node for the
same string during one execution, sub-queries included.

Collections and containers (RML-CC). The head of a collection without
expression is a blank node per iteration (and per combination of values for
`rml:cartesianProduct`); a named collection has the term of its expression as
head, and the elements of all the iterations that give the same head, in the same
graph, form one collection, in the order of the iterations. The rank of an
element is the number of elements of its collection with a smaller order key:
building a collection is quadratic in its size. An empty collection is `rdf:nil`, and an
empty container a node with its type only, when `rml:allowEmptyListAndContainer` is
true; otherwise neither is generated. Not translated: `rml:cartesianProduct` in a
nested gather map (its members are appended), a gather map in a predicate map
or a graph map. A referencing object map among the members, without join
condition, gives the subject of its parent evaluated on the same iteration.

Functions (RML-FNML). Each function known to the translator is an expression of
SPARQL or of the function library of SPARQL-Generate (`iterators/functions.rqg`),
its inputs substituted by their values:

| function | expression |
|---|---|
| `grel:toUpperCase`, `grel:toLowerCase`, `grel:toTitlecase` | `UCASE`, `LCASE`, `fun:titleCase` |
| `grel:string_trim`, `grel:string_length` | `REPLACE(..., "^\\s+\|\\s+$", "")`, `STRLEN` |
| `grel:string_substring` (`p_int_i_from`, `p_int_i_opt_to`) | `SUBSTR`, no value past the end |
| `grel:string_replace` (`param_find`, `param_replace`) | `REPLACE(..., "q")`, both strings literal |
| `grel:string_contains`, `grel:string_startsWith`, `grel:string_endsWith` | `CONTAINS`, `STRSTARTS`, `STRENDS` |
| `grel:string_md5`, `grel:string_sha1` | `MD5`, `SHA1` |
| `grel:escape` (`modeParam`: `html`, `xml`, `url`) | `REPLACE` of `&`, `<`, `>`, `"`; `ENCODE_FOR_URI` |
| `idlab-fn:equal`, `idlab-fn:notEqual`, `idlab-fn:concat`, `idlab-fn:trueCondition`, `idlab-fn:toUpperCaseURL` | `=`, `!=`, `CONCAT`, `IF`, `UCASE` with `HTTP://` |

(`grel:` is `http://users.ugent.be/~bjdmeest/function/grel.ttl#`, `idlab-fn:`
`https://w3id.org/imec/idlab/function#`.) Another function, a parameter that its
function does not have, an output that it does not return, are `#ERROR` lines. Not
translated: the functions of other libraries (a Java implementation given
to an RML processor has no SPARQL counterpart, unless one writes it as a
SPARQL-Generate `FUNCTION`), the GREL functions on arrays (`array_join`,
`string_split`), functions with several outputs. The default term type of a
function-valued object map is `rml:Literal`, with the datatype of the value of
the expression (`xsd:integer` for `string_length`).

Logical views (RML-LV). A view iteration is a solution of the iterators of
its fields, so an expression field without value removes the iteration (RML-LV
keeps it, with a null value); a CSV source gives the index key `#` from the
position of the row (SPARQL-Generate 2.4.2, as for gather maps), and an iterable field over
CSV text gives the index of its rows likewise. Fields nested deeper than three levels are not
checked by the key checks. Cycles (`rml:viewOn`, joins, fields), two fields with the
same name, and a reference to a key that the view does not have (or to an iterable
field, or to `<it>`) are `#ERROR` lines. The joins of a view that is itself the
source of another view, or the parent of a join, are not translated.

RML-star. The quoted triples map has one predicate-object map, with one
predicate and one object. Star maps are translated over CSV sources (the
references of the quoted triples map are columns of the same row, or of the
joined row); over JSONPath and XPath sources, only constant quoted triples
maps. The output writes triple terms in the RDF 1.2 syntax of Jena,
`<<( s p o )>>`, in subject position too (RDF-star quoted triples, `<< s p o >>`).
A `rml:quotedTriplesMap` outside a subject map or an object map is an `#ERROR`.

Sources and targets (RML-IO). A compressed source (`rml:compression`: gzip, zip,
tar.gz, tar.xz) and a source in UTF-16 with a byte order mark (`rml:encoding`) are
read by SPARQL-Generate 2.4.2 or later, which decompresses local files by the
extension of their name and transcodes them to UTF-8; 2.4.1 reads only gzip. Not
translated: logical targets (`rml:logicalTarget`, `rml:LogicalTarget`,
`rml:target`, serialization, compression, encoding of the output): a
SPARQL-Generate query has one output, written by the command line; a target
would need an output construct (a file per `GRAPH` block, or an output
clause), which the language does not have. Nor sources that are not files
(databases, SPARQL endpoints, a SPARQL query over a file:
`formats:SPARQL_Results_*`), XPath with namespaces
(`rml:XPathReferenceFormulation`, `rml:namespace`) or with the parent axis
(`..`, the iteration being the serialised element), an encoding without byte
order mark, the empty CSV cell as a value (SPARQL-Generate reads it as no value),
and the errors of a CSV file with rows of the wrong length.

## Optimised translation

The generated query is the direct translation, followed by mechanical rewritings; each is exact (the same output up to isomorphism), and is checked
by the conformance suites (no case changes status) and by the benchmark workloads (the
same distinct triples):

| | rewriting | when |
|---|---|---|
| R1 | the sub-queries of an iteration (one for the classes, one per predicate map and object map) are flattened into the WHERE clause of the triples map: each term is bound once per row, and the triples map has one template | CSV sources: a reference has one value per row, so each sub-query has at most one solution |
| R2 | `IRI(?l)` instead of a run-time test for a relative IRI | templates whose constant text starts with a scheme |
| R3 | the triples maps of one CSV file share one iteration of it: one GENERATE with all their triples, `iter:CSV` over the union of their columns | the same file and reference formulation, each triples map with at least one reference or template, and no blank node map without expression |
| R4 | CSV files are read in batches of 10 000 rows (a streamed output is written batch by batch) | always: the generated queries have no solution modifier |

Gather maps over a CSV source need SPARQL-Generate 2.4.2 or later: the order of
the rows is the position that `iter:CSV` gives after the named columns (on an
earlier version, the iteration has no solution). Over JSONPath and XPath sources, 2.4.0
is enough.

The translation needs SPARQL-Generate 2.4.0 or later: graph maps are translated into
`GRAPH` blocks of the `GENERATE` template (quad patterns). The query generated from a
mapping without graph maps has no `GRAPH` block, and also runs on 2.2.3 and later.

## Conformance

Measured by kgc-bench with SPARQL-Generate 2.4.0 before its release (branch
`quad-patterns`, commit `8ba16cf`), by RDF dataset isomorphism with the expected output of each case, the default
graph and every named graph (an expected error passes when an error is raised, with
`--fail-on-missing`):

| suite | cases | PASS | FAIL |
|---|---|---|---|
| RMLio/rml-test-cases@803dd3ec, CSV | 39 | 38 | 1 |
| RMLio/rml-test-cases@803dd3ec, JSON | 40 | 39 | 1 |
| RMLio/rml-test-cases@803dd3ec, XML | 38 | 37 | 1 |
| kg-construct/rml-core@82ab28d4, JSON | 76 | 73 | 3 |

The suites of the RML modules (KG-Construct), measured by kgc-bench (`9b4215a`) on
SPARQL-Generate 2.4.1 and on the branch `rml-modules` of SPARQL-Generate (2.4.2-SNAPSHOT,
commit `6fb62b9`), which reads the row position of `iter:CSV` and compressed or
UTF-16 files:

| suite | cases | PASS on 2.4.1 | PASS on `rml-modules` |
|---|---|---|---|
| kg-construct/rml-cc@d97fddf | 35 | 33 | 35 |
| kg-construct/rml-fnml@f573f79 | 22 | 21 | 21 |
| kg-construct/rml-lv@94beb94 | 41 | 36 | 40 |
| kg-construct/rml-star@711f72e | 18 | 18 | 18 |
| kg-construct/rml-io@0cb26a2 | 73 | 20 | 24 |

rml-cc and rml-lv need the row position for their CSV cases (gather maps over CSV,
the index key `#`). The rml-fnml failure is a reference to a column that the CSV file
does not have, an error that the generated query does not raise (as RML-Core 0002c);
the rml-lv one an inner join with a parent iteration whose field has no value (0010b).
rml-io: 23 of its 32 source cases pass; its 41 target cases (`rml:logicalTarget`), but
one, fail: the output is not written to the target files (see above).

The cases with graph maps (6 per track of rml-test-cases, 7 of rml-core) all pass, as do
the cases of kgc-bench (`conformance/extra`) for graph maps in a shared CSV iteration, in
joins, and with JSONPath values in number other than one. The failures are errors that
only the data reveal (an unknown column or file, a reference to an array, an invalid
IRI), which the generated query does not raise, and `rml:UnsafeIRI` values with spaces,
which the engine rejects as IRIs (Jena's `IRI()`, also behind `<...{?x}>`). A missing
file is an error with `--fail-on-missing` (2.2.3).

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
