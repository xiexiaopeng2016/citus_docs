.. _querying:

查询分布式表（SQL)
=========================

正如前面部分所讨论的，Citus是一个扩展，它扩展了最新的PostgreSQL以实现分布式执行。
这意味着您可以在Citus协调者上使用标准的PostgreSQL `SELECT <http://www.postgresql.org/docs/current/static/sql-select.html>`_ 查询进行查询。
然后，Citus将并行化涉及复杂选择，分组和排序以及JOIN的SELECT查询，以加快查询性能。
在较高的层次上，Citus将SELECT查询分区为较小的查询片段，将这些查询片段分配给工作者节点，监督其执行，合并其结果（并在需要时对其进行排序)，并将最终结果返回给用户。

在以下部分中，我们将讨论可以使用Citus运行的不同类型的查询。

.. _aggregate_functions:

聚合函数
-------------

Citus支持和并行化PostgreSQL支持的大多数聚合函数。Citus的查询规划器将聚合转换为其可交换和关联形式，因此可以并行化。在此过程中，工作者节点在分片上运行聚合查询，然后协调者将工作者节点的结果组合在一起以生成最终输出。

.. _count_distinct:

Count(Distinct) 聚合
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Citus以多种方式支持count(distinct)聚合。如果count(distinct)聚合在分发列上，Citus可以直接将查询下推到工作者节点。如果没有，Citus会在每个工作者节点上运行select distinct语句，并将列表返回给协调者，在协调者中获取最终计数。

请注意，当工作人员拥有更多不同的项目时，传输此数据会变慢。对于包含多个count(distinct)聚合的查询尤其如此，例如：

.. code-block:: sql

  -- 一个查询中的多个distinct counts 往往是缓慢的
  SELECT count(distinct a), count(distinct b), count(distinct c)
  FROM table_abc;


对于这类查询，在工作者节点上生成的select distinct语句本质上生成要传输给协调者的行的交叉乘积。

为了提高性能，您可以选择进行近似计数。请按照以下步骤操作：

1. 在所有PostgreSQL实例（协调者和所有工作者)上下载并安装hll扩展。

   请访问PostgreSQL hll `github存储库 <https://github.com/citusdata/postgresql-hll>`_ 以获取有关获取扩展的详细信息。

2. 在所有PostgreSQL实例上创建hll扩展

  .. code-block:: postgresql

    CREATE EXTENSION hll;

3. 通过设置Citus.count_distinct_error_rate配置值启用计数不同的近似值。此配置设置的较低值预计会提供更准确的结果，但需要更多时间进行计算。我们建议将其设置为0.005。

  .. code-block:: postgresql

    SET citus.count_distinct_error_rate to 0.005;

  在此步骤之后，count(distinct)聚合会自动切换到使用HLL，而不需要对查询进行任何更改。您应该能够在表的任何列上运行近似count distinct查询。

HyperLogLog列
$$$$$$$$$$$$$

某些用户已将其数据存储为HLL列。在这种情况下，他们可以通过调用hll_union_agg（hll_column)动态地汇总这些数据。

.. _topn:

估计前N项
~~~~~~~~~

通过应用计数，排序和限制来计算集合中的前*n*个元素很简单。但是，随着数据大小的增加，此方法变得缓慢且资源密集。使用近似值更有效。

Postgres的开源 `TopN 扩展 <https://github.com/citusdata/postgresql-topn>`_ 可以快速获得“top-n”查询的近似结果。扩展将top值实现为JSON数据类型。TopN可以逐步更新这些最高值，或者在不同的时间间隔内按需合并它们。

**基本操作**

在看到一个真实的TopN示例之前，让我们看看它的一些原始操作是如何工作的。首先 ``topn_add`` 更新一个JSON对象，其中包含一个被看到的次数的键：

.. code-block:: postgres

  -- starting from nothing, record that we saw an "a"
  select topn_add('{}', 'a');
  -- => {"a": 1}

  -- record the sighting of another "a"
  select topn_add(topn_add('{}', 'a'), 'a');
  -- => {"a": 2}

该扩展还提供聚合以扫描多个值：

.. code-block:: postgres

  -- for normal_rand
  create extension tablefunc;

  -- count values from a normal distribution
  SELECT topn_add_agg(floor(abs(i))::text)
    FROM normal_rand(1000, 5, 0.7) i;
  -- => {"2": 1, "3": 74, "4": 420, "5": 425, "6": 77, "7": 3}

如果不同值的数量超过阈值，则聚合会丢弃最不常见的信息。这可以控制空间使用。阈值可以由``topn.number_of_counters` GUC 控制。其默认值为1000。

**现实的例子**

现在谈谈TopN如何在实践中运作的更现实的例子。让我们从2000年开始收集亚马逊产品评论，并使用TopN快速查询。首先下载数据集：

.. code-block:: bash

  curl -L https://examples.citusdata.com/customer_reviews_2000.csv.gz | \
    gunzip > reviews.csv

接下来，将其摄取到分布式表中：

.. code-block:: psql

  CREATE TABLE customer_reviews
(
      customer_id TEXT,
      review_date DATE,
      review_rating INTEGER,
      review_votes INTEGER,
      review_helpful_votes INTEGER,
      product_id CHAR(10),
      product_title TEXT,
      product_sales_rank BIGINT,
      product_group TEXT,
      product_category TEXT,
      product_subcategory TEXT,
      similar_product_ids CHAR(10)[]
  );

  SELECT create_distributed_table('customer_reviews', 'product_id');

  \COPY customer_reviews FROM 'reviews.csv' WITH CSV

接下来我们将添加扩展，创建一个目标表来存储由TopN生成的json数据，并应用我们之前看到的函数 ``topn_add_agg``。

.. code-block:: postgresql

  -- note: Citus Cloud has extension already
  CREATE EXTENSION topn;
  SELECT run_command_on_workers(' create extension topn; ');

  -- a table to materialize the daily aggregate
  CREATE TABLE reviews_by_day
(
    review_date date unique,
    agg_data jsonb
  );

  SELECT create_reference_table('reviews_by_day');

  -- materialize how many reviews each product got per day per customer
  INSERT INTO reviews_by_day
    SELECT review_date, topn_add_agg(product_id)
    FROM customer_reviews
    GROUP BY review_date;

现在，我们不需要在 ``customer_reviews`` 上编写复杂的窗口函数，只需将TopN应用于 ``reviews_by_day`` 即可。例如，以下查询查找前五天中每个最常查看的产品：

.. code-block:: postgres

  SELECT review_date,(topn(agg_data, 1)).*
  FROM reviews_by_day
  ORDER BY review_date
  LIMIT 5;

::

  ┌───────────────┬──────┬───────────┐
  │ review_date │    item     │ frequency  │
  ├───────────────┼──────┼───────────┤
  │ 2000-01-01  │ 0939173344  │        12  │
  │ 2000-01-02  │ B000050XY8  │        11  │
  │ 2000-01-03  │ 0375404368  │        12  │
  │ 2000-01-04  │ 0375408738  │        14  │
  │ 2000-01-05  │ B00000J7J4  │        17  │
  └───────────────┴──────┴───────────┘


TopN创建的json字段可以与 ``topn_union`` 和 ``topn_union_agg`` 合并。我们可以使用后者来合并整个第一个月的数据，并列出在此期间最受关注的五个产品。

.. code-block:: postgres

  SELECT(topn(topn_union_agg(agg_data), 5)).*
  FROM reviews_by_day
  WHERE review_date >= '2000-01-01' AND review_date < '2000-02-01'
  ORDER BY 2 DESC;

::

  ┌────────────┬───────────┐
  │    item    │ frequency │
  ├─────────────┼──────────┤
  │ 0375404368 │       217 │
  │ 0345417623 │       217 │
  │ 0375404376 │       217 │
  │ 0375408738 │       217 │
  │ 043936213X │       204 │
  └─────────────┴──────────┘

有关更多详细信息和示例，请参阅 `TopN 自述文件 <https://github.com/citusdata/postgresql-topn/blob/master/README.md>`_。

.. _limit_pushdown:

限制下推
-------------

Citus还尽可能地将limit子句下放到工作者节点的分片上，以最大限度地减少通过网络传输的数据量。

但是，在某些情况下，使用LIMIT子句的SELECT查询可能需要从每个分片中获取所有行以生成精确结果。例如，如果查询需要按聚合列排序，则需要所有分片中该列的结果来确定最终聚合值。由于大量的网络数据传输，这降低了LIMIT子句的性能。在这种情况下，当近似将产生有意义的结果时，Citus为网络有效近似LIMIT子句提供了一个选项。

默认情况下LIMIT近似值是禁用的，可以通过设置配置参数citus.limit_clause_row_fetch_count来启用LIMIT近似值。根据这个配置值，Citus将限制每个任务返回的行数，以便在协调者上进行聚合。由于此限制，最终结果可能是近似值。增加这个限制将提高最终结果的准确性，同时仍然提供从工作者节点中提取的行数的上限。

.. code-block:: postgresql

    SET citus.limit_clause_row_fetch_count to 10000;

分布式表的视图
-------------------

Citus支持分布式表的所有视图。有关视图语法和功能的概述，请参阅 `CREATE VIEW <https://www.postgresql.org/docs/current/static/sql-createview.html>`_ 的PostgreSQL文档。

请注意，某些视图导致查询计划效率低于其他视图。有关检测和改善较差视图性能的更多信息，请参阅 :ref:`subquery_perf`。（视图在内部被视为子查询。)

Citus也支持物化视图，并将它们作为本地表存储在协调者节点上。在实现之后在分布式查询中使用它们需要将它们包装在子查询中，这是一种在 :ref:`join_local_dist` 中描述的技术。

.. _joins:

Joins
-----

Citus支持任意数量的表之间的等连接，而不管它们的大小和分布方法。查询规划器根据表的分布方式选择最佳连接方法和连接顺序。它评估几个可能的连接顺序，并创建一个连接计划，该计划要求跨网络传输的数据最少。

Co-located joins
~~~~~~~~~~~~~~~~

当两个表 :ref:`位于同一位置 <colocation>` 时，它们可以在其共同的分布列上高效地连接。位于同一位置的连接是连接两个大型分布式表的最有效方式。

在内部，Citus协调者通过查看分发列元数据来了解位于同一位置的表的哪些分片可能与另一个表的分片匹配。这允许Citus修剪掉不能产生匹配的连接键的分片对。剩余分片对之间的连接在工作者节点上并行执行，然后将结果返回给协调者。

.. note::

  确保将表分布到相同数量的分片中，并确保每个表的分发列具有完全匹配的类型。尝试连接稍微不同类型的列（如int和bigint)可能会导致问题。

引用表连接
~~~~~~~~~~

:ref:`reference_tables` 可以用作“维度”表，以便与大的“事实”表有效地连接。由于引用表在所有工作者节点上完全复制，因此引用连接分解可以为每个工作程序上的本地连接并并行执行。引用连接类似于位于同一位置连接的更灵活版本，因为引用表不分布在任何特定列上，并且可以在任何列上自由加入。

.. _repartition_joins:

重新分区连接
~~~~~~~~~~~~~~~~~~

在某些情况下，您可能需要在分发列以外的列上连接两个表。对于这种情况，Citus还允许通过对查询的表进行动态重新分区来连接非分布键列。

在某些情况下，查询优化器根据分布列、连接键和表的大小来确定要分区的表。使用重新分区的表，可以确保只有相关的分片对相互连接，从而大大减少跨网络传输的数据量。

通常，位于同一位置的连接比重新分区连接更有效，因为重新分区连接需要重排数据。因此，您应该尝试尽可能通过位于同一位置的连接键来分布表。
