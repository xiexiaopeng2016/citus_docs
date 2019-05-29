.. _citus_sql_reference:

SQL支持和解决方法
=================

由于Citus通过扩展PostgreSQL提供分布式功能，所以它与PostgreSQL结构兼容。这意味着用户可以将丰富且可扩展的PostgreSQL生态系统附带的工具和功能用于使用Citus创建的分布式表。

对于能够在单个工作节点上执行的任何查询，Citus具有100％的SQL覆盖率。
这些类型的查询称为:ref:`router executable <router_executor>`，在访问有关单个租户的信息时，在:ref:`mt_use_case`中很常见。

甚至跨节点查询（用于并行计算）也支持大多数SQL功能。但是，对于组合来自多个节点的信息的查询，不支持某些SQL特性。

**跨节点SQL查询的限制:**

* `窗口函数 <https://www.postgresql.org/docs/current/static/tutorial-window.html>`_仅在PARTITION BY中包含分布列时才支持。
* S`SELECT … FOR UPDATE <https://www.postgresql.org/docs/current/static/sql-select.html#SQL-FOR-UPDATE-SHARE>`_ 仅在:ref:`router_executor`查询中工作
* `TABLESAMPLE <https://www.postgresql.org/docs/current/static/sql-select.html#SQL-FROM>`_仅在:ref:`router_executor`查询中工作
* 仅当关联位于:ref:`dist_column`上且子查询符合子查询下推规则（例如，通过分发列进行分组，没有LIMIT或LIMIT OFFSET子句）时，才支持相关子查询。
* `递归 CTEs <https://www.postgresql.org/docs/current/static/queries-with.html#idm46428713247840>`_仅在:ref:`router_executor`查询中起作用
* `Grouping sets <https://www.postgresql.org/docs/current/static/queries-table-expressions.html#QUERIES-GROUPING-SETS>`_仅适用于:ref:`router_executor`查询

要了解有关PostgreSQL及其功能的更多信息，可以访问`PostgreSQL文档<http://www.postgresql.org/docs/current/static/index.html>`_。有关PostgreSQL SQL命令方言（Citus用户可以使用）的详细参考，您可以看到`SQL命令参考 <http://www.postgresql.org/docs/current/static/sql-commands.html>`_。

.. _workarounds:

解决方法
--------

在尝试变通方法之前，请考虑Citus是否适合您的情况。Citus的当前版本适用于:ref:`real-time analytics and multi-tenant use cases. <when_to_use_citus>`。

Citus支持多租户用例中的所有SQL语句。即使在实时分析用例中，通过跨节点的查询，Citus也支持大多数语句。几种不支持的查询类型列在:ref:`unsupported`中，许多不支持的特性都有解决方案;下面是一些最有用的。Citus不支持任何PostgreSQL功能，列出了几种不支持的查询？许多不受支持的功能都有解决方法; 以下是一些最有用的。

.. _join_local_dist:

连接本地和分布式表
~~~~~~~~~~~~~~~~~

尝试在本地表"local"和分布式表"dist"之间执行JOIN会导致错误：

.. code-block:: sql

  SELECT * FROM local JOIN dist USING (id);

  /*
  ERROR:  relation local is not distributed
  STATEMENT:  SELECT * FROM local JOIN dist USING (id);
  ERROR:  XX000: relation local is not distributed
  LOCATION:  DistributedTableCacheEntry, metadata_cache.c:711
  */

虽然您无法直接连接此类表，但通过将本地表包装在子查询或CTE中，可以使Citus的递归查询计划程序将本地表数据复制到工作节点。通过共置数据，这允许查询继续进行

.. code-block:: sql

  -- either

  SELECT *
    FROM (SELECT * FROM local) AS x
    JOIN dist USING (id);

  -- or

  WITH x AS (SELECT * FROM local)
  SELECT * FROM x
  JOIN dist USING (id);

请记住，协调者会将子查询或CTE中的结果发送给需要进行处理的所有工作者。
因此，最好是尽可能向内部查询添加最特定的过滤器和限制，或者聚合表。
这减少了这种查询可能导致的网络开销。有关:ref:`subquery_perf`的更多信息。

时表:最后的解决方案
~~~~~~~~~~~~~~~~~~

即使通过子查询使用推拉执行，仍有一些查询:ref:`不受支持 <unsupported>`。其中一个是运行由非分布列分区的窗口函数。

假设我们有一个名为的表:code:`github_events`，由列:code:`user_id`分布。那么以下窗口函数将不起作用

.. code-block:: sql

  -- this won't work

  SELECT repo_id, org->'id' as org_id, count(*)
    OVER (PARTITION BY repo_id) -- repo_id is not distribution column
    FROM github_events
   WHERE repo_id IN (8514, 15435, 19438, 21692);

还有另外一个技巧。我们可以将相关信息作为临时表提供给协调者：

.. code-block:: sql

  -- grab the data, minus the aggregate, into a local table

  CREATE TEMP TABLE results AS (
    SELECT repo_id, org->'id' as org_id
      FROM github_events
     WHERE repo_id IN (8514, 15435, 19438, 21692)
  );

  -- now run the aggregate locally

  SELECT repo_id, org_id, count(*)
    OVER (PARTITION BY repo_id)
    FROM results;

在协调者上创建临时表是最后的手段。它受节点的磁盘大小和CPU的限制。

