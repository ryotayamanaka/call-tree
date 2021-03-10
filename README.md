# call-tree

This repository is for evaluating that **Oracle Graph Server** is useful for the call tree analysis. [Oracle Graph Server and Client](https://www.oracle.com/database/technologies/spatialandgraph/property-graph-features/graph-server-and-client.html) is a part of Oracle Database features, based on an Oracle Labs' technology, [Parallel Graph AnalytiX (PGX)](https://www.oracle.com/middleware/technologies/parallel-graph-analytix.html). This evaluation is influenced by the discussion on GraalVM below.

- [Call tree analysis with graph databases (e.g. Neo4j) v2](https://github.com/oracle/graal/pull/3128)

## Environment

**Oracle Graph Server 21.1** is available in the options below:

- [RPM](https://www.oracle.com/database/technologies/spatialandgraph/property-graph-features/graph-server-and-client/graph-server-and-client-downloads.html) downloaded and installed on your environment, **or**
- [Marketplace image](https://cloudmarketplace.oracle.com/marketplace/en_US/listing/75067377) launched on Oracle Cloud

The scripts below are tested on Oracle Cloud Compute instance (VM.Standard2.1 = 1 CPU / 15 GB memory). PGX is run in the stand-alone mode (not the server mode) and loading the graph data from files (not from relational database) here.

## Loading

The data source is the CSV files in [the sample report folder](https://www.dropbox.com/s/z0s6adzg27wf3g4/reports-csv-1112.tgz?dl=0) linked from [this issue comment](https://github.com/oracle/graal/pull/2957#issuecomment-743175407). The mapping from CSV files (= tables) to graph is defined in the loading configuration file [`config.json`](./config.json).

Please note that one of the input files (`*_entry_points_*.csv`) is modified to include the VM IDs, so the n:m relationships can be properly loaded from this file, while this ID is always 0 in this case: 
```sh
sed 's/^/0,/' csv_call_tree_entry_points_helloworld_20201211_112253.csv \
| sed '1s/0,Id/VMId,MethodId/' > csv_call_tree_entry_points_helloworld_20201211_112253_mod.csv
```

`*_entry_points_*.csv` (original)
```
Id
0
1
2
```


`*_entry_points_*_mod.csv` (modified)
```
VMId,MethodId
0,0
0,1
0,2
```


Please also note that the `Method` nodes are loaded from two CSV files, and the sum of them (8863 = 7509 + 1354) is shown in the output below. You can find the numbers of nodes and edges are the same in [the outputs here](https://github.com/oracle/graal/pull/2957#issuecomment-756227414).

### JShell

Start JShell:
```
$ opg-jshell
13:09:25,159 INFO Ctrl$1 - >>> start engine
For an introduction type: /help intro
Oracle Graph Server Shell 21.1.0
Variables instance, session, and analyst ready to use.
opg-jshell>
```

Load the dataset into a graph and count the numbers of nodes and edges:
```java
opg-jshell> var graph = session.readGraphWithProperties("./config.json")
graph ==> PgxGraph[name=call_tree,N=8864,E=26906,created=1615122575811]

opg-jshell> graph.queryPgql(" SELECT LABEL(v), COUNT(v) FROM MATCH (v) GROUP BY LABEL(v) ").print()
+---------------------+
| LABEL(v) | COUNT(v) |
+---------------------+
| VM       | 1        |
| Method   | 8863     |
+---------------------+

opg-jshell> graph.queryPgql(" SELECT LABEL(e), COUNT(e) FROM MATCH ()-[e]->() GROUP BY LABEL(e) ").print()
+-------------------------+
| LABEL(e)     | COUNT(e) |
+-------------------------+
| virtual      | 5533     |
| direct       | 18554    |
| overriden_by | 2819     |
+-------------------------+

/exit
```

### Python

Start Python shell:
```
$ opgpy
Oracle Graph Server Shell 21.1.0
>>>
```

Load the dataset into a graph and count the numbers of nodes and edges:
```python
>>> graph = session.read_graph_with_properties("./config.json")
>>> graph
PgxGraph(name: call_tree, v: 8864, e: 26906, directed: True, memory(Mb): 5)

>>> graph.query_pgql(" SELECT LABEL(v), COUNT(v) FROM MATCH (v) GROUP BY LABEL(v) ").print()
+---------------------+
| LABEL(v) | COUNT(v) |
+---------------------+
| VM       | 1        |
| Method   | 8863     |
+---------------------+

>>> graph.query_pgql(" SELECT LABEL(e), COUNT(e) FROM MATCH ()-[e]->() GROUP BY LABEL(e) ").print()
+-------------------------+
| LABEL(e)     | COUNT(e) |
+-------------------------+
| virtual      | 5533     |
| direct       | 18554    |
| overriden_by | 2819     |
+-------------------------+
```


### Performance

Measure the loading time of the graph:
```python
import time
start = time.time()
graph = session.read_graph_with_properties("./config.json")
end = time.time()
print(end - start)
```

Output:
```
2.9235193729400635
```

The loading time (10 times) were between 2.5 and 3.5.

## Queries

Find the all methods connecting to a particular method.
```sql
SELECT LABEL(e), n.name
FROM MATCH (m:method)<-[e]-(n:method)
WHERE m.name = 'getAndBitwiseOrInt'
```
```
+----------------------------+
| LABEL(e) | name            |
+----------------------------+
| direct   | getAndBitwiseOr |
+----------------------------+
```

Find the all methods connecting to a particular method **with exactly 4 hops** and the relationships are `direct` or `virtual`.
```sql
PATH any AS () -[:direct|virtual]-> ()
SELECT n.name
FROM MATCH (m:method)<-/:any{4}/-(n:method)
WHERE m.name = 'getAndBitwiseOrInt'
```
```
+---------------------+
| name                |
+---------------------+
| doInvoke            |
| tryRemoveAndExec    |
| helpCC              |
| topLevelExec        |
| invoke              |
| propagateCompletion |
| awaitJoin           |
| tryComplete         |
+---------------------+
```

Find the all methods connecting to a particular method **in 1 to 4 hops** and the relationships are `direct` or `virtual`.
```sql
PATH any AS () -[:direct|virtual]-> ()
SELECT n.name
FROM MATCH (m:method)<-/:any{1,4}/-(n:method)
WHERE m.name = 'getAndBitwiseOrInt'
```
```
+---------------------+
| name                |
+---------------------+
| getAndBitwiseOr     |
| setDone             |
| externalAwaitDone   |
| internalWait        |
| doExec              |
| quietlyComplete     |
| doInvoke            |
| awaitJoin           |
| topLevelExec        |
| helpCC              |
| tryRemoveAndExec    |
| propagateCompletion |
| tryComplete         |
| invoke              |
+---------------------+
```

## Appendix: Original Vertex IDs

The original IDs of the vertices, which are connected by the edges, can be shown using the queries below:

```sql
-- direct edges
SELECT ID(e), s.id AS s_id, d.id AS d_id, LABEL(e), e.bci
FROM MATCH (s)-[e:direct]->(d)
ORDER BY s_id, d_id ASC LIMIT 10

-- virtual edges
SELECT ID(e), s.id AS s_id, d.id AS d_id, LABEL(e), e.bci
FROM MATCH (s)-[e:virtual]->(d)
ORDER BY s_id, d_id ASC LIMIT 10

-- overriden_by edges
SELECT ID(e), s.id AS s_id, d.id AS d_id, LABEL(e)
FROM MATCH (s)-[e:overriden_by]->(d)
ORDER BY s_id, d_id ASC LIMIT 10
```
