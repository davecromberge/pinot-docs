---
description: Troubleshoot issues with the multi-stage query engine (v2).
---

# Troubleshoot issues with the multi-stage query engine (v2)

Learn how to [troubleshoot errors](troubleshoot-multi-stage-query-engine.md#troubleshoot-errors) when using the multi-stage query engine (v2), and see [multi-stage query engine limitations](troubleshoot-multi-stage-query-engine.md#multi-stage-query-engine-limitations).&#x20;

Find instructions on [how to enable the multi-stage query engine](v2-multi-stage-query-engine.md), or see a [high-level overview](https://app.gitbook.com/o/-LtRX9NwSr7Ga7zA4piL/s/-LtH6nl58DdnZnelPdTc-887967055/\~/changes/1760/reference/cluster-1) of how the multi-stage query engine works.

## Troubleshoot errors

Troubleshoot [cluster errors](troubleshoot-multi-stage-query-engine.md#cluster-errors), [semantic/runtime errors](troubleshoot-multi-stage-query-engine.md#semantic-runtime-errors), and [timeout errors](troubleshoot-multi-stage-query-engine.md#timeout-errors).

### Cluster errors

If you can't get the cluster to run and receive an error indicating the multi-stage engine is not available, do the following:

* Verify your Pinot servers have access to the configured port. By default, Pinot assumes all servers are run on individual networks, and all of them have access to the same port number available within that environment. If your cluster is not set up this way, you will have to manually configure your ports directly in your pinot-server config file.

### Semantic/runtime errors

* Try downloading the latest docker image or building from the latest master commit.
  * We continuously push bug fixes for the multi-stage engine so bugs you encountered might have already been fixed in the latest master build.
* Try rewriting your query.
  * Some functions previously supported in the single-stage query engine (v1) may have a new way to express in the multi-stage engine (v2). Check and see if you are using any non-standard SQL functions or semantics.

### Timeout errors

* Try reducing the size of the table(s) used.&#x20;
  * Add higher selectivity filters to the tables.
* Try executing part of the subquery or a simplified version of the query first.
  * This helps to determine the selectivity and scale of the query being executed.
* Try adding more servers.
  * The new multi-stage engine runs distributed across the entire cluster, so adding more servers to partitioned queries such as GROUP BY aggregates, and equality JOINs help speed up the query runtime.

## Multi-stage query engine limitations

We are continuously improving the v2 multi-stage query engine. A few limitations to call out:

### Support for multi value columns is limited

Support for multi value columns is limited to projections, and predicates must use the `arrayToMv` function. For example, to successfully run the following query:

{% code overflow="wrap" %}
```sql
SELECT count(*), RandomAirports FROM airlineStats GROUP BY RandomAirports
```
{% endcode %}

You must include `arrayToMv` in the query as follows:

{% code overflow="wrap" %}
```sql
SELECT count(*), RandomAirports FROM airlineStats GROUP BY arrayToMv(RandomAirports)
```
{% endcode %}

### Schema and other prefixes are not supported

Schema and other prefixes are not supported in queries. For example, the following queries are _**not**_ supported:

```
SELECT* from default.myTable;
SELECT * from schemaName.myTable;
```

&#x20;Queries _**without prefixes are supported**_:&#x20;

```
SELECT * from myTable;
```

### Modifying query behavior based on the cluster config is not supported

Modifying query behavior based on the cluster configuration is not supported. `distinctcounthll`, `distinctcounthllmv`, `distinctcountrawhll`, and \```distinctcountrawhllmv` use`` a different default value of `log2mParam` in the multi-stage v2 engine. In v2, this value can no longer be configured. Therefore, the following query may produce different results in v1 and v2 engine:

```sql
select distinctcounthll(col) from myTable
```

To ensure v2 returns the same result, specify the `log2mParam` value in your query:

```sql
select distinctcounthll(col, 8) from myTable
```

### ORDER BY with duplicate columns

&#x20;In v2, referencing a column twice with`ORDER BY` is not supported. For example, the following query will fail: &#x20;

```sql
SELECT ArrTime, ArrTime FROM mytable order by ArrTime
```

To successfully run this query, include an alias as follows:

```sql
SELECT ArrTime as c1, ArrTime FROM mytable order by c1
```

### Tightened restriction on function naming

Pinot single-stage query engine automatically removes the underscore `_ character from function names. So co_u_n_t()`is equivalent to `count().`

In v2, function naming restrictions were tightened, so the underscore(`_)` character is only allowed to separate word boundaries in a function name. Also camel case is supported in function names. For example, the following function names are allowed:

```markup
is_distinct_from(...)
isDistinctFrom(...)
```

### Default names for projections with function calls

Default names for projections with function calls are different between v1 and v2.&#x20;

* For example, in v1, the following query:

```sql
SELECT count(*) from mytable 
```

Returns the following result:

```
    "columnNames": [
        "EXPR$0"
      ],
```

* In v2, the following function:

```sql
SELECT count(*) from mytable
```

&#x20;Returns the following result:

```
      "columnNames": [
        "count(*)"
      ],
```

### Table names and column names are case sensitive

In v2, table and column names and are case sensitive. In v1 they were not. For example, the following two queries are not equivalent in v2:

`select * from myTable`

`select * from mytable`

{% hint style="info" %}
**Note:** Function names are not case sensitive in v2 or v1.
{% endhint %}

### Arbitrary number of arguments isn't supported

An arbitrary number of arguments is no longer supported in v2. For example, in v1, the following query worked:

<pre><code><a data-footnote-ref href="#user-content-fn-1">select add(1,2,3,4,5) from table</a>
</code></pre>

In v2, this query must be rewritten as follows:

```
select add(1, add(2,add(3, add(4,5)))) from table
```

### NULL function support





###

[^1]: 