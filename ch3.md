# Chapter 3  Query Processing

As described in the [official document](https://www.postgresql.org/docs/current/static/features.html), PostgreSQL supports a very large number of features required by the SQL standard of 2011. Query processing is the most complicated subsystem in PostgreSQL, and it efficiently processes the supported SQL. This chapter outlines this query processing; in particular, it focuses on query optimization.

This chapter comprises the following three parts:

- **Part 1:** Section 3.1.
- This section overviews the query processing in PostgreSQL.

- **Part 2:** Sections 3.2. — 3.4.
- This part explains the steps followed to obtain the optimal plan of a single-table query. In Sections 3.2 and 3.3, the processes of estimating the cost and creating the plan tree are explained, respectively. In Section 3.4, the operation of the executor is briefly described.

- **Part 3:** Sections 3.5. — 3.6.
- This part explains the process of obtaining the optimal plan of a multiple-table query. In Section 3.5, three join methods are described: nested loop, merge and hash join. In Section 3.6, the process of creating the plan tree of a multiple-table query is explained.

PostgreSQL supports three technically interesting and practical features, namely, [Foreign Data Wrappers (FDW)](http://www.postgresql.org/docs/current/static/fdwhandler.html), [Parallel Query](https://www.postgresql.org/docs/current/static/parallel-query.html) and [JIT compilation](https://www.postgresql.org/docs/11/static/jit-reason.html) which is supported from version 11. The first two of them will be described in [Chapter 4](http://www.interdb.jp/pg/pgsql04.html). The JIT compilation is out of scope of this document; see the [official document](https://www.postgresql.org/docs/11/static/jit-reason.html) in details.

## 3.1. Overview

In PostgreSQL, although the parallel query implemented in version 9.6 uses multiple background worker processes, a backend process basically handles all queries issued by the connected client. This backend consists of five subsystems, as shown below:

1. Parser
2. The parser generates a parse tree from an SQL statement in plain text.

3. Analyzer/Analyser
4. The analyzer/analyser carries out a semantic analysis of a parse tree and generates a query tree.

5. Rewriter
6. The rewriter transforms a query tree using the rules stored in the [rule system](http://www.postgresql.org/docs/current/static/rules.html) if such rules exist.

7. Planner
8. The planner generates the plan tree that can most effectively be executed from the query tree.

9. Executor
10. The executor executes the query via accessing the tables and indexes in the order that was created by the plan tree.

**Fig. 3.1. Query Processing.**

![Fig. 3.1. Query Processing.](http://www.interdb.jp/pg/img/fig-3-01.png)![img]()

In this section, an overview of these subsystems is provided. Due to the fact that the planner and the executor are very complicated, a detailed explanation for these functions will be provided in the following sections.



PostgreSQL's query processing is described in the [official document](http://www.postgresql.org/docs/current/static/overview.html) in detail.



### 3.1.1. Parser

The parser generates a parse tree that can be read by subsequent subsystems from an SQL statement in plain text. Here a specific example is shown in the following without a detailed description.

Let us consider the query shown below.

```sql-monosp
testdb=# SELECT id, data FROM tbl_a WHERE id < 300 ORDER BY data;
```

A parse tree is a tree whose root node is the [SelectStmt](javascript:void(0)) structure defined in [parsenodes.h](https://github.com/postgres/postgres/blob/master/src/include/nodes/parsenodes.h). Figure 3.2(b) illustrates the parse tree of the query shown in Fig. 3.2(a).

**Fig. 3.2. An example of a parse tree.**

![Fig. 3.2. An example of a parse tree.](http://www.interdb.jp/pg/img/fig-3-02.png)![img]()

The elements of the SELECT query and the corresponding elements of the parse tree are numbered the same. For example, (1) is an item of the first target list and it is the column ‘id’ of the table, (4) is a WHERE clause, and so on.

Due to the fact that the parser only checks the syntax of an input when generating a parse tree, it only returns an error if there is a syntax error in the query.

The parser does not check the semantics of an input query. For example, even if the query contains a table name that does not exist, the parser does not return an error. Semantic checks are done by the analyzer/analyser.

### 3.1.2. Analyzer/Analyser

The analyzer/analyser runs a semantic analysis of a parse tree generated by the parser and generates a query tree.

The root of a query tree is the [Query](javascript:void(0)) structure defined in [parsenodes.h](https://github.com/postgres/postgres/blob/master/src/include/nodes/parsenodes.h); this structure contains metadata of its corresponding query such as the type of this command (SELECT, INSERT or others) and several leaves; each leaf forms a list or a tree and holds data of the individual particular clause.

Figure 3.3 illustrates the query tree of the query shown in Fig. 3.2(a) in the previous subsection.

**Fig. 3.3. An example of a query tree.**

![Fig. 3.3. An example of a query tree.](http://www.interdb.jp/pg/img/fig-3-03.png)![img]()

The above query tree is briefly described as follows.

- The targetlist is a list of columns that are the result of this query. In this example, this list is composed of two columns: *‘id'* and *‘data’*. If the input query tree uses ‘**\(\ast\)**' (asterisk), the analyzer/analyser will explicitly replace it to all of the columns.
- The range table is a list of relations that are used in this query. In this example, this table holds the information of the table ‘*tbl_a*’ such as the *oid* of this table and the name of this table.
- The join tree stores the FROM clause and the WHERE clauses.
- The sort clause is a list of SortGroupClause.

The details of the query tree are described in the [official document](http://www.postgresql.org/docs/current/static/querytree.html).

### 3.1.3. Rewriter

The rewriter is the system that realizes the [rule system](http://www.postgresql.org/docs/current/static/rules.html), and transforms a query tree according to the rules stored in the [pg_rules](http://www.postgresql.org/docs/current/static/view-pg-rules.html) system catalog if necessary. The rule system is an interesting system in itself, however, the descriptions of the rule system and the rewriter have been omitted to prevent this chapter from becoming too long.



 *View*

[Views](https://www.postgresql.org/docs/current/static/rules-views.html) in PostgreSQL are implemented by using the rule system. When a view is defined by the [CREATE VIEW](http://www.postgresql.org/docs/current/static/sql-createview.html) command, the corresponding rule is automatically generated and stored in the catalog.

Assume that the following view is already defined and the corresponding rule is stored in the *pg_rules* system catalog.

```sql-monosp
sampledb=# CREATE VIEW employees_list 
sampledb-#      AS SELECT e.id, e.name, d.name AS department 
sampledb-#            FROM employees AS e, departments AS d WHERE e.department_id = d.id;
```

When a query that contains a view shown below is issued, the parser creates the parse tree as shown in Fig. 3.4(a).

```sql-monosp
sampledb=# SELECT * FROM employees_list;
```

At this stage, the rewriter processes the range table node to a parse tree of the subquery, which is the corresponding view, stored in *pg_rules*.

**Fig. 3.4. An example of the rewriter stage.**

![Fig. 3.4. An example of the rewriter stage.](http://www.interdb.jp/pg/img/fig-3-04.png)![img]()

Since PostgreSQL realizes views using such a mechanism, views could not be updated until version 9.2. However, views can be updated from version 9.3 onwards; nonetheless, there are many limitations in updating the view. These details are described in the [official document](https://www.postgresql.org/docs/current/static/sql-createview.html#SQL-CREATEVIEW-UPDATABLE-VIEWS).



### 3.1.4. Planner and Executor

The planner receives a query tree from the rewriter and generates a (query) plan tree that can be processed by the executor most effectively.

The planner in PostgreSQL is based on pure cost-based optimization; it does not support rule-based optimization and hints. This planner is the most complex subsystem in RDBMS; therefore, an overview of the planner will be provided in the subsequent sections of this chapter.



 *pg_hint_plan*

PostgreSQL does not support the planner hints in SQL, and it will not be supported forever. If you want to use hints in your queries, the extension referred to *pg_hint_plan* will be worth considering. Refer to the [official site](http://pghintplan.osdn.jp/pg_hint_plan.html) in detail.



As in the other RDBMS, the [EXPLAIN](http://www.postgresql.org/docs/current/static/sql-explain.html) command in PostgreSQL displays the plan tree itself. A specific example is shown below.

```sql-monosp
testdb=# EXPLAIN SELECT * FROM tbl_a WHERE id < 300 ORDER BY data;
                          QUERY PLAN                           
---------------------------------------------------------------
 Sort  (cost=182.34..183.09 rows=300 width=8)
   Sort Key: data
   ->  Seq Scan on tbl_a  (cost=0.00..170.00 rows=300 width=8)
         Filter: (id < 300)
(4 rows)
```

This result shows the plan tree shown in Fig. 3.5.

**Fig. 3.5. A simple plan tree and the relationship between the plan tree and the result of the EXPLAIN command.**

![Fig. 3.5. A simple plan tree and the relationship between the plan tree and the result of the EXPLAIN command.](http://www.interdb.jp/pg/img/fig-3-05.png)![img]()

A plan tree is composed of elements called *plan nodes*, and it is connected to the plantree list of the *PlannedStmt* structure. These elements are defined in [plannodes.h](https://github.com/postgres/postgres/blob/master/src/include/nodes/plannodes.h). Details will be explained in Section 3.3.3 (and Section 3.5.4.2).

Each plan node has information that the executor requires for processing, and the executor processes from the end of the plan tree to the root in the case of a single-table query.

For example, the plan tree shown in Fig. 3.5 is a list of a sort node and a sequential scan node; thus, the executor scans the table:*tbl_a* by a sequential scan and then sorts the obtained result.

The executor reads and writes tables and indexes in the database cluster via the buffer manager described in [Chapter 8](http://www.interdb.jp/pg/pgsql08.html). When processing a query, the executor uses some memory areas, such as temp_buffers and work_mem, allocated in advance and creates temporary files if necessary.

In addition, when accessing tuples, PostgreSQL uses the concurrency control mechanism to maintain consistency and isolation of the running transactions. The concurrency control mechanism is described in [Chapter 5](http://www.interdb.jp/pg/pgsql05.html).

**Fig. 3.6. The relationship among the executor, buffer manager and temporary files.**

![Fig. 3.6. The relationship among the executor, buffer manager and temporary files.](http://www.interdb.jp/pg/img/fig-3-06.png)![img]()

 

------

## 3.2. Cost Estimation in Single-Table Query

PostgreSQL's query optimization is based on cost. Costs are dimensionless values, and these are not absolute performance indicators but are indicators to compare the relative performance of operations.

Costs are estimated by the functions defined in [costsize.c](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c). All of operations executed by the executor have the corresponding cost functions. For example, the costs of sequential scans and index scans are estimated by cost_seqscan() and cost_index(), respectively.

In PostgreSQL, there are three kinds of costs: **start-up**, **run** and **total**. The total cost is the sum of start-up and run costs; thus, only the start-up and run costs are independently estimated.

- The **start-up** cost is the cost expended before the first tuple is fetched. For example, the start-up cost of the index scan node is the cost to read index pages to access the first tuple in the target table.
- The **run** cost is the cost to fetch all tuples.
- The **total** cost is the sum of the costs of both start-up and run costs.

The [EXPLAIN](https://www.postgresql.org/docs/current/static/sql-explain.html) command shows both of start-up and total costs in each operation. The simplest example is shown below:

```sql-monosp
testdb=# EXPLAIN SELECT * FROM tbl;
                       QUERY PLAN                        
---------------------------------------------------------
 Seq Scan on tbl  (cost=0.00..145.00 rows=10000 width=8)
(1 row)
```

In Line 4, the command shows information about the sequential scan. In the cost section, there are two values; 0.00 and 145.00. In this case, the start-up and total costs are 0.00 and 145.00, respectively.

In this section, we explore how to estimate the sequential scan, index scan and sort operation in detail.

In the following explanations, we use a specific table and an index that are shown below:

```sql-monosp
testdb=# CREATE TABLE tbl (id int PRIMARY KEY, data int);
testdb=# CREATE INDEX tbl_data_idx ON tbl (data);
testdb=# INSERT INTO tbl SELECT generate_series(1,10000),generate_series(1,10000);
testdb=# ANALYZE;
testdb=# \d tbl
      Table "public.tbl"
 Column |  Type   | Modifiers 
--------+---------+-----------
 id     | integer | not null
 data   | integer | 
Indexes:
    "tbl_pkey" PRIMARY KEY, btree (id)
    "tbl_data_idx" btree (data)
```

### 3.2.1. Sequential Scan

The cost of the sequential scan is estimated by the cost_seqscan() function. In this subsection, we explore how to estimate the sequential scan cost of the following query.

```sql-monosp
testdb=# SELECT * FROM tbl WHERE id < 8000;
```

In the sequential scan, the start-up cost is equal to 0, and the run cost is defined by the following equation:

‘run cost’=‘cpu run cost’+‘disk run cost’=(cpu_tuple_cost+cpu_operator_cost)×Ntuple+seq_page_cost×Npage,

where [seq_page_cost](https://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-SEQ-PAGE-COST), [cpu_tuple_cost](https://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-CPU-TUPLE-COST) and [cpu_operator_cost](https://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-CPU-OPERATOR-COST) are set in the postgresql.conf file, and the default values are *1.0*, *0.01*, and *0.0025*, respectively; \(N_{tuple}\) and \(N_{page}\) are the numbers of all tuples and all pages of this table, respectively, and these numbers can be shown using the following query:

```sql-monosp
testdb=# SELECT relpages, reltuples FROM pg_class WHERE relname = 'tbl';
 relpages | reltuples 
----------+-----------
       45 |     10000
(1 row)
```

\begin{align} N_{tuple} &= 10000, \tag{1} \\ N_{page} &= 45. \tag{2} \\ \end{align}

Thus,

\begin{align} \verb|‘run cost’| &= (0.01 + 0.0025) \times 10000 + 1.0 \times 45 = 170.0. \end{align}

Finally,

\begin{align} \verb|‘total cost’| = 0.0 + 170.0 = 170. \end{align}

For confirmation, the result of the EXPLAIN command of the above query is shown below:

```sql-monosp
testdb=# EXPLAIN SELECT * FROM tbl WHERE id < 8000;
                       QUERY PLAN                       
--------------------------------------------------------
 Seq Scan on tbl  (cost=0.00..170.00 rows=8000 width=8)
   Filter: (id < 8000)
(2 rows)
```

In Line 4, we can find that the start-up and total costs are 0.00 and 170.00, respectively, and it is estimated that 8000 rows (tuples) will be selected by scanning all rows.

In Line 5, a filter ‘Filter:(id < 8000)’ of the sequential scan is shown. More precisely, it is called a *table level filter predicate*. Note that this type of filter is used when reading all the tuples in the table, and it does not narrow the scanned range of table pages.



As understood from the run-cost estimation, PostgreSQL assumes that all pages will be read from storages; that is, PostgreSQL does not consider whether the scanned page is in the shared buffers or not.



### 3.2.2. Index Scan

Although PostgreSQL supports [some index methods](https://www.postgresql.org/docs/current/static/indexes-types.html), such as BTree, [GiST](https://www.postgresql.org/docs/current/static/gist.html), [GIN](https://www.postgresql.org/docs/current/static/gin.html) and [BRIN](https://www.postgresql.org/docs/current/static/brin.html), the cost of the index scan is estimated using the common cost function: cost_index().

In this subsection, we explore how to estimate the index scan cost of the following query:

```sql-monosp
testdb=# SELECT id, data FROM tbl WHERE data < 240;
```

Before estimating the cost, the numbers of the index pages and index tuples, \(N_{index,page}\) and \(N_{index,tuple}\), are shown below:

```sql-monosp
testdb=# SELECT relpages, reltuples FROM pg_class WHERE relname = 'tbl_data_idx';
 relpages | reltuples 
----------+-----------
       30 |     10000
(1 row)
```

\begin{align} N_{index,tuple} &= 10000, \tag{3} \\ N_{index,page} &= 30. \tag{4} \end{align}

#### 3.2.2.1. Start-Up Cost

The start-up cost of the index scan is the cost to read the index pages to access to the first tuple in the target table, and it is defined by the following equation:

\begin{align} \verb|‘start-up cost’| = \{\mathrm{ceil}(\log_2 (N_{index,tuple})) + (H_{index} + 1) \times 50\} \times \verb|cpu_operator_cost|, \end{align}

where \(H_{index}\) is the height of the index tree.

In this case, according to (3), \(N_{index,tuple}\) is 10000; \(H_{index}\) is 1; \(\verb|cpu_operator_cost|\) is 0.0025 (by default). Thus,

\begin{align} \verb|‘start-up cost’| = \{\mathrm{ceil}(\log_2(10000)) + (1 + 1) \times 50\} \times 0.0025 = 0.285. \tag{5} \end{align}

#### 3.2.2.2. Run Cost

The run cost of the index scan is the sum of the cpu costs and the IO (input/output) costs of both the table and the index:

\begin{align} \verb|‘run cost’| &= (\verb|‘index cpu cost’| + \verb|‘table cpu cost’|) + (\verb|‘index IO cost’| + \verb|‘table IO cost’|). \end{align}

If the [Index-Only Scans](https://www.postgresql.org/docs/10/static/indexes-index-only-scans.html), which is described in [Section 7.2](http://www.interdb.jp/pg/pgsql07.html#_7.2.), can be applied, \(\verb|‘table cpu cost’|\) and \(\verb|‘table IO cost’|\) are not estimated.



The first three costs (i.e. index cpu cost, table cpu cost and index IO cost) are shown in below:

\begin{align} \verb|‘index cpu cost’| &= \verb|Selectivity| \times N_{index,tuple} \times (\verb|cpu_index_tuple_cost| + \verb|qual_op_cost|), \\ \verb|‘table cpu cost’| &= \verb|Selectivity| \times N_{tuple} \times \verb|cpu_tuple_cost|, \\ \verb|‘index IO cost’| &= \mathrm{ceil}(\verb|Selectivity| \times N_{index,page}) \times \verb|random_page_cost|, \end{align}

where [cpu_index_tuple_cost](https://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-CPU-INDEX-TUPLE-COST) and [random_page_cost](https://www.postgresql.org/docs/current/static/runtime-config-query.html#GUC-RANDOM-PAGE-COST) are set in the postgresql.conf file (the defaults are 0.005 and 4.0, respectively); \(\verb|qual_op_cost|\) is, roughly speaking, the evaluating cost of the index, and it is shown without much explanation here: \(\verb|qual_op_cost| = 0.0025\). \(\verb|Selectivity|\) is the proportion of the search range of the index by the specified WHERE clause; it is a floating point number from 0 to 1, and it is described in detail in below. For example, \((\verb|Selectivity| \times N_{tuple})\) means the *number of the table tuples to be read*, \((\verb|Selectivity| \times N_{index,page})\) means the *number of the index pages to be read* and so on.



 *Selectivity*

The selectivity of query predicates is estimated using histogram_bounds or the MCV (Most Common Value), both of which are stored in the statistics information [pg_stats](https://www.postgresql.org/docs/current/static/view-pg-stats.html). Here, the calculation of the selectivity is briefly described using specific examples. More details are provided in the [official document](https://www.postgresql.org/docs/10/static/row-estimation-examples.html).

The MCV of each column of a table is stored in the [pg_stats](https://www.postgresql.org/docs/10/static/view-pg-stats.html) view as a pair of columns of *most_common_vals* and *most_common_freqs*.

- *most_common_vals* is a list of the MCVs in the column.
- *most_common_freqs* is a list of the frequencies of the MCVs.

A simple example is shown as follows. The table ["countries"](javascript:void(0)) has two columns: a column ‘country’ that stores the country name and a column ‘continent’ that stores the continent name to which the country belongs.