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
| term types `IRI` (IRI-safe encoding, exact on non-ASCII), `URI`, `UnsafeIRI`, `UnsafeURI`, `Literal`, `BlankNode` | `IRI(...)` with base-IRI concatenation; `UnsafeIRI` and `UnsafeURI`: `fun:unsafeIRI(...)` (2.6.1), the string as it is, unchecked (it may hold spaces); `STRLANG`, `STRDT`, `fun:bnode(...)` |
| `rr:language`, `rr:datatype`, `rml:languageMap`, `rml:datatypeMap` | `STRLANG`, `STRDT` |
| `rr:class`, shortcuts `rr:subject`/`rr:predicate`/`rr:object`/`rr:graph` | yes |
| referencing object map, same logical source, no join | the parent's subject, on the same iteration |
| referencing object map with `rr:joinCondition` (`rml:childMap`/`rml:parentMap` too) | the parent is generated once as a graph (`FROM GENERATE`), the join is a basic graph pattern over it; for a CSV child whose subject is a blank node without expression, the join is a sub-query in the child's iteration (the blank node of a row must be the same in both) |
| `rml:baseIRI` (per triples map) | yes |
| graph maps: `rr:graphMap`, `rr:graph`, `rml:graphMap`, `rml:graph` (RML-Core), on the subject map (every triple of the triples map, `rr:class` included) and on each predicate-object map; several graph maps: the triple in each; `rr:defaultGraph`: the default graph | the default graph outside `GRAPH` blocks, `GRAPH <g> { ... }` for a constant, `GRAPH ?g { ... }` for a template or a reference (term type IRI), bound once per row (CSV) or in a sub-query of the predicate-object map, one quad per value (JSONPath, XPath); with a join, the graph term is carried from the child's iteration |
| gather maps (RML-CC, `http://w3id.org/rml/`): `rml:gather`, `rml:gatherAs` (`rdf:List`, `rdf:Bag`, `rdf:Seq`, `rdf:Alt`), `rml:strategy` (`rml:append`, `rml:cartesianProduct`), `rml:allowEmptyListAndContainer`, in a subject map or an object map, nested gather maps, named collections (`rml:template`, `rml:reference`, `rml:constant`, IRI or blank node), members that are referencing object maps, with graph maps | one sub-query per gather map at the top level: a `FROM GENERATE` iterates over the logical source and records each element of each collection with an order key (position of the iteration, member, positions of the values), then the cells (`rdf:first`, `rdf:rest`, `rdf:_n`) are built from the rank of each element in its collection (see below) |
| function-valued term maps (RML-FNML): `rml:functionExecution`, `rml:function` or `rml:functionMap`, `rml:input` with `rml:parameter` or `rml:parameterMap`, `rml:inputValue` or `rml:inputValueMap` (reference, template, constant, function: composition), `rml:return` or `rml:returnMap`; in any term map, language and datatype maps included; `rml:condition` | the SPARQL expression of the function (see below), in place of the lexical form of the term map; a condition gives the term only when it is true |
| a triples map without logical source, of constant term maps (RML-FNML) | a sub-query without iterator, evaluated once |
| logical views (RML-LV): `rml:LogicalView`, `rml:viewOn` (a logical source or another view), `rml:field` (`rml:ExpressionField` with `rml:reference`, `rml:template` or `rml:constant`; `rml:IterableField` with `rml:iterator`; nested fields, `rml:referenceFormulation` to change formulation: JSON in CSV, CSV in JSON), the index keys `#` and `<field>.#`, `rml:leftJoin` and `rml:innerJoin` (`rml:parentLogicalView`, `rml:joinCondition` with `rml:child`/`rml:parent` or `rml:childMap`/`rml:parentMap`, joined fields) | the iteration of the view as binding clauses: one iterator per field, nested over the records of its parent (one solution per logical view iteration; a JSONPath expression field without value gives one iteration where it is null), each key bound to a variable that the term maps read as a CSV column; a join is a `FROM GENERATE` of the parent view's iterations, generated once, in which each iteration of the child looks up its key, bound beforehand (`OPTIONAL` for a left join; the joined fields `OPTIONAL`, a null field staying unbound; a child key without value is an IRI, which no key equals): the join costs the iterations of the child and their matches, not their product |
| RML-star: `rml:quotedTriplesMap` in a subject map or an object map (star maps), on the same logical source or with `rml:joinCondition`, nested quoted triples, `rml:AssertedTriplesMap`, `rml:NonAssertedTriplesMap` | the triple term `afn:triple(s, p, o)` (Jena, RDF 1.2) of the terms of the quoted triples map on the same iteration; with a join, a `FROM GENERATE` of the quoted triples of the other source, matched in the `WHERE` clause; a non-asserted triples map generates no triple of its own |
| RML-IO sources: `rml:source` as a literal path, `rml:FilePath` and `rml:RelativePathSource` (`rml:path`, `rml:root`); `rml:null` | the file, relative to the base IRI; a value among the null values of the source is no value (CSV) |
| RML-IO CSV sources (`http://w3id.org/rml/`) | `ITERATOR iter:CSVStrict(<file>, ...)` (SPARQL-Generate 2.6.1): an empty cell is the empty string, a row with another number of fields than the header is a data error |
| RML-IO SPARQL results: `formats:SPARQL_Results_CSV` (`_JSON`, `_XML`, `_TSV`), the iterator a `SELECT` query over an RDF file | `ITERATOR iter:SPARQL(<file>, """<query>""", "var", ...)` (2.6.1): SPARQL-Generate runs the query itself, one row per solution, the references naming its variables |
| relational databases: an R2RML logical table (`rr:logicalTable`, `rr:tableName`, `rr:sqlQuery`) or a logical source on a D2RQ database (`d2rq:Database`, `d2rq:jdbcDSN`, `d2rq:username`, `d2rq:password`) with `rr:tableName`, `rml:query`, or the RML-IO formulations `rml:SQL2008Table` and `rml:SQL2008Query` (the iterator is the table or the query); `rr:sqlVersion rr:SQL2008` | `ITERATOR iter:SQL("<JDBC URL with the user and the password; for MySQL and a query in SQL:2008, the ANSI_QUOTES mode, where double quotes delimit identifiers>", """<effective SQL query>""", "col", ...) AS ?c_...` (2.6.1): `SELECT * FROM <table>` (the name as written, a SQL identifier) or the query; the columns are the natural RDF literals of R2RML (`xsd:integer`, `xsd:decimal`, `xsd:double`, `xsd:boolean`, `xsd:date`, `xsd:time`, `xsd:dateTime`, `xsd:hexBinary`) for a literal without datatype or language, their lexical forms in templates; a `NULL` gives no term; an unknown column, a duplicate column name, a query that the database rejects stop the execution. A logical table without database uses the only database of the mapping |
| RML-IO XPath with namespaces (`rml:XPathReferenceFormulation`, `rml:namespace`) | as `rml:XPath`: the prefixes are those that the document declares |
| XPath references that leave the iterated element (`../@id`, `ancestor::`, sibling axes) | sub-queries of the iteration, `ITERATOR iter:XPath(<file>, "<iterator>", "../@id") AS ?it ?xp_...`, evaluated from the element in its document (2.6.1) |
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
position of the row (SPARQL-Generate 2.5.0, as for gather maps), and an iterable field over
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
read by SPARQL-Generate 2.5.0 or later, which decompresses local files by the
extension of their name and transcodes them to UTF-8; 2.4.1 reads only gzip. A SPARQL query over a file (`formats:SPARQL_Results_*`), XPath with namespaces
and with references that leave the iterated element, the empty CSV cell as a value
and the errors of a CSV file with rows of the wrong length are translated for
SPARQL-Generate 2.6.1 (`iter:SPARQL`, `iter:XPath` sub-queries, `iter:CSVStrict`;
the mapping is first read as a graph where an XPath formulation with namespaces is
`rml:XPath` and SPARQL results are `rml:CSV`). Relational databases are read by
`iter:SQL` (2.6.1, which embeds the JDBC drivers of PostgreSQL, MySQL and SQL Server). Not
translated: SPARQL endpoints, namespace prefixes that the XML document does not
declare itself, and an encoding without byte order mark.

Logical targets (RML-IO, SPARQL-Generate 2.6.0 or later). A triple whose subject map,
predicate map, object map, or language or datatype map has `rml:logicalTarget` is
written in the file of each of its targets instead of the output, in each of its graphs;
a graph map with `rml:logicalTarget` sends the quads of its graph to the files of its
targets too. The file is a named graph whose IRI is the path of the target
(`rml:target [ a rml:FilePath; rml:path "dump1.nq" ]`, relative to the base, whatever
its `rml:root`): `GRAPH <dump1.nq> { ... }`, and a graph `g` inside the file is
`GRAPH <dump1.nq#g=ENCODE_FOR_URI(g)> { ... }`. The command line writes these graphs to
their files with `--graphs-to-files` (format and compression by extension), or with
`--targets targets.json`, the description of the targets that the second query of this
repository, `rml2targets.rqg`, generates from the mapping (`rml:serialization`,
`rml:compression`, `rml:encoding`, UTF-16 for instance):

```sh
(cd work && java -jar sparql-generate.jar -d . -q rml2targets.rqg -o targets.json)
java -jar sparql-generate.jar -d data -q generated.rqg -b http://example.com/base/ \
     --targets work/targets.json -fo NQUADS
```

The files are written in the working directory (`-d`), which `-b` identifies with the
base, or in the folder of a `file:` base.

Data errors (SPARQL-Generate 2.6.0 or later). The generated query raises the errors
that only the data reveal with `RAISE`, an expression of SPARQL-Generate 2.6 that stops
the execution with a message (status 1 on the command line):

- a reference to a column that the header of a CSV source does not have: one sub-query
  per CSV triples map, before the triples maps, reads the header only
  (`iter:CSVHeaders(<file>)`) and raises the error for a missing column
  (`templates/columnChecks.rqg`; RML-Core and RML 1.x);
- RML 1.x: a reference of a JSON or XML source that has no value in any record (the
  counterpart of the missing column): one sub-query per triples map reads the document from
  its root and evaluates the iterator followed by each reference for a first match
  (`templates/keyChecks.rqg`; not for a JSONPath iterator without wildcard, filter or deep
  scan);
- RML-Core: a JSONPath reference whose value is an array or an object, and the value of
  a reference that is not a valid IRI for a term map of term type IRI or URI. RML 1.x
  omits these terms, as its test cases expect.

A reference that is absent from some JSON or XML records only is no value in those records,
not an error; RML-Core does not raise the error of the absent reference either (its
test cases no longer expect it, kg-construct/rml-io-registry#46).

## Optimised translation

The generated query is the direct translation, followed by mechanical rewritings; each is exact (the same output up to isomorphism), and is checked
by the conformance suites (no case changes status) and by the benchmark workloads (the
same distinct triples):

| | rewriting | when |
|---|---|---|
| R1 | the sub-queries of an iteration (one for the classes, one per predicate map and object map) are flattened into the WHERE clause of the triples map: each term is bound once per row, and the triples map has one template | CSV sources: a reference has one value per row, so each sub-query has at most one solution |
| R2 | `IRI(?l)` instead of a run-time test for a relative IRI | templates whose constant text starts with a scheme |
| R3 | the triples maps of one CSV file share one iteration of it: one GENERATE with all their triples, `iter:CSV` over the union of their columns | the same file and reference formulation, each triples map with at least one reference or template, no blank node map without expression, and no null values of its own (`rml:null`) |
| R4 | CSV files are read in batches of 10 000 rows (a streamed output is written batch by batch) | always: the generated queries have no solution modifier |
| R8 | a join of a CSV source with itself is made on the row of the child, in its iteration (the parent's subject computed from the same row, as for a referencing object map without join), instead of two key graphs (`FROM GENERATE`) | the child and the parent read the same CSV file (the same logical source, or logical sources of the same description: same source, formulation and properties; not a view, nor a database); every join condition joins a column with itself (the join columns c: one, or several, a composite key); the child's subject map or the parent's subject map is an IRI template (without `\`) whose references are all join columns, each at least once, or the reference of the only join column (the parent's then without function nor condition; the other subject map: the child's an IRI template or reference, the parent's with an expression); the child's predicates and graph maps are constants; no function, star map, gather map nor logical target on the child's subject map, the parent's subject map or the referencing object map. A child subject on c: the subject (and predicate, graphs) of a child row depends on its values of c only, so a pair of rows with the same values gives the triple of the parent row paired with itself, and a row without one of the values has no subject. A parent subject on c (the symmetric case): every parent row with the values of c of a child row has the same subject, that of the child row itself, which is one of them; a row without one of the values has no parent subject and joins no row (an empty key matches no key; RML-IO reads an empty cell as the empty string, a value, which the row then joins with itself, as the direct translation does). No uniqueness of c is needed (GTFS-Madrid-Bench: shapes and their points, services and their calendar rules; the symmetric case: the points of a shape and the shape, in gtfs-inverted; with the feed as a second join column, the concatenated feeds of GTFS France) |
| R9 | the child of a join is read as a stream, at the level of the join: each row binds its key and looks it up in the graph of the parent keys (`FROM GENERATE`), instead of a second graph of the child keys | a CSV child (not a logical view), join conditions given as references (`rr:child`, `rr:parent`), one or several (a composite key, below), a constant predicate, no graph map with an expression, and a parent subject that is an IRI template whose references are all among the parent's references of the conditions (at least one), or one of them. Exact in any case (the same parent graph; a child row generates the triples of its key; an empty key, bound to an IRI, matches no key, and a composite key with an empty component is empty); the condition on the parent subject bounds the cost: a child row finds at most one parent subject, and the join costs the rows of the child |

A join with several conditions (`(feed_id, trip_id)`, a referencing object map or a join of
logical views) compares one composite key: the values of its conditions, as strings, each
preceded by its length and `:` (`"1:A2:r1"`), an encoding without collision whatever the
values hold; a missing value of any condition leaves the key unbound, so the row joins
nothing. The key is looked up at once: with one key per condition, joined one condition after
the other, the join costs every pair of child and parent rows with the same value of the
first condition (the feed), the next tested afterwards: a quadratic join within each feed.

The time of the translation grows with the size of the mapping: the GTFS-Madrid-Bench
mapping once per feed, 100 feeds and 1 300 triples maps, translates in about 80 s with
SPARQL-Generate 2.6.3 (0.5.3 and earlier: over 600 s). Each query of the translator follows
the mapping from the nodes it is called with; a query that would read the whole mapping for
every triples map (a pattern joined to its parameters only after its evaluation, a path whose
first step has two free ends) costs the square of the number of triples maps.

Gather maps over a CSV source need SPARQL-Generate 2.5.0 or later: the order of
the rows is the position that `iter:CSV` gives after the named columns (on an
earlier version, the iteration has no solution). Over JSONPath and XPath sources, 2.4.0
is enough.

The translation needs SPARQL-Generate 2.6.1 or later for a mapping of the CSV sources of
RML-Core and RML-IO (`http://w3id.org/rml/`), SPARQL results and XPath references that
leave the iterated element (`iter:CSVStrict`, `iter:SPARQL`, relative `iter:XPath`
sub-queries) and for relational databases (`iter:SQL` with named columns and natural
literals), and 2.6.0 or later otherwise: the generated queries use `RAISE` (data
errors) and `iter:CSVHeaders` on a file. With 2.6.0, use the translator 0.3.0. With 2.4.0 to 2.5.0, use the translator
at commit `5fa291a`. SPARQL-Generate 2.6.2 or later is recommended for large sources: it
reads a CSV or JSON file in batches when the output does not depend on them, and generates
the joins of R9 over the graph of the parent keys (`FROM GENERATE`) as they are read, so that
the translated GTFS-Madrid-Bench mapping runs at scale 1000 in less than 10 GB.
Graph maps are translated into
`GRAPH` blocks of the `GENERATE` template (quad patterns), as are logical targets.

## Conformance

Measured by kgc-bench with SPARQL-Generate 2.6.3 and pysparql-generate 0.4.2, the same case by case (the SQL
tracks with 2.6.2 and the translator 0.5.2), by RDF dataset isomorphism with the
expected output of each case, the default graph and every named graph; the logical
targets are written to their files (`--graphs-to-files`, `--targets`) and compared with
the expected `dumpN.*` files; an expected error passes when an error is raised (with
`--fail-on-missing`):

| suite | cases | PASS | FAIL |
|---|---|---|---|
| RMLio/rml-test-cases@803dd3ec (CSV, JSON, XML) | 117 | 117 | 0 |
| kg-construct/rml-core@82ab28d4 | 76 | 76 | 0 |
| kg-construct/rml-io@0cb26a2 | 73 | 72 | 1 |
| RMLio/rml-test-cases@803dd3ec, SQL tracks (PostgreSQL, MySQL, SQL Server) | 60 + 60 + 60 | 57, 59, 58 | 3, 1, 2 |
| kg-construct/rml-fnml@f573f79 | 22 | 22 | 0 |
| kg-construct/rml-cc@d97fddf | 35 | 35 | 0 |
| kg-construct/rml-lv@94beb94 | 41 | 41 | 0 |
| kg-construct/rml-star@9517932 | 18 | 18 | 0 |

Every target case of rml-io passes (the 41 RMLTTC cases, UTF-16 with `--targets`); its
failure is SPARQL results whose expected output types the plain literals of the file as
`xsd:integer` (0003: the query runs, the literals stay as they are). The D2RQ database of 0006a and the SQL tracks of rml-test-cases
run on PostgreSQL, MySQL and SQL Server containers started by kgc-bench (each case loads its
`resource.sql`). Their failures are those of the cases: 0002f (an error expected for references that
differ from the column names in case only; the original R2RML case creates delimited
columns `"ID"`, `"Name"`, the RML port creates them unquoted, so they match as regular SQL
identifiers on every database), 0015a on PostgreSQL (the query names a table `"Country"`
that the script creates unquoted, `country` for PostgreSQL; on MySQL, the ANSI_QUOTES mode
of a query in SQL:2008 makes it pass), 0016e on PostgreSQL and SQL Server (the script stores
the text of the hexadecimal digits, not the bytes: `'\\x89...'`, `CAST('89...' AS
VARBINARY)`). The generated queries raise the data errors with `RAISE`: an unknown CSV
column (rml-test-cases 0002c-CSV, rml-fnml 0101), a JSON or XML reference without value in
any record (rml-test-cases 0002c-JSON, 0002c-XML, as RMLMapper, FlexRML and BURP), a value
that is not an IRI (rml-core 0019b), a reference to an array (rml-core 0025b). The
`rml:UnsafeIRI` values with spaces of rml-core 0027b are written as they are
(`fun:unsafeIRI`, 2.6.1; N-Triples escapes the space as `\u0020`), although RML-Core's
term generation rules make an invalid IRI a data error (kg-construct/rml-core#72, #265).
A JSONPath field without value of a logical view keeps the iteration with a null value
(rml-lv 0010b).

The cases with graph maps (6 per track of rml-test-cases, 7 of rml-core) all pass, as do
the cases of kgc-bench (`conformance/extra`) for graph maps in a shared CSV iteration, in
joins, and with JSONPath values in number other than one, and for the joins of rewritings R8
and R9 (a CSV file joined with itself, logical sources of the same description or not, a
child or a parent subject that depends on the join column only, a duplicate row, a child read
as a stream, empty keys), and for the left and inner joins of logical views (empty keys on
both sides, two matches, a null field of the parent; X012), and for joins on a composite key (two
conditions, a missing value of either, values whose plain concatenations collide, several
matches: X013 for referencing object maps, read as a stream, made on the same row or not;
X014 for logical views).

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
