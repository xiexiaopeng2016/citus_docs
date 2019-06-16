.. _citus_query_processing:

查询处理
=============

Citus集群由一个协调者实例和多个工作者实例组成。当协调者存储有关这些分片的元数据时，数据被分片并复制到工作者上。向集群发出的所有查询都通过协调者执行。协调者将查询划分为较小的查询片段，其中每个查询片段都可以在一个分片上独立运行。然后协调者将查询片段分配给工作者，监视它们的执行，合并它们的结果，并将最终结果返回给用户。查询处理体系结构可以通过下图进行简要描述。

.. image:: ../images/citus-high-level-arch.png

Citus的查询处理管道包括两个组件:

* **分布式查询规划器和执行器**
* **PostgreSQL查询规划器和执行器**

我们将在后续章节中更详细地讨论它们。

.. _distributed_query_planner:

分布式查询规划器
----------------------

Citus的分布式查询规划器接受SQL查询并规划分布式执行。

对于SELECT查询，计划程序首先创建输入查询的计划树，并将其转换为其可交换和关联形式，以便可以并行化。它还应用了多种优化，以确保以可伸缩的方式执行查询，并最大限度地减少网络I / O.

接下来，规划器将查询分为两部分 - 在协调者上运行的协调者查询和在工作者上的各个分片上运行的工作程序查询片段。然后，规划器将这些查询片段分配给工作者，以便有效地使用它们的所有资源。在此步骤之后，将分布式查询计划传递给分布式执行器执行

对于在分布列上的键值查找或修改查询的规划过程略有不同，因为它们只命中一个切分。规划器一旦接收到传入查询，就需要决定将查询路由到正确的切分。分发列或修改查询上的键值查找的规划过程略有不同，因为它们只针对一个分片。计划程序收到传入查询后，需要确定应将查询路由到的正确分片。为此，它将提取传入行中的分发列，并查找元数据以确定查询的正确分片。然后，规划器重写该命令的SQL以引用分片表而不是原始表。然后将此重写的计划传递给分布式执行程序。

.. _distributed_query_executor:

分布式查询执行器
---------------------


Citus的分布式执行器运行分布式查询计划，并处理查询执行过程中发生的故障。执行程序连接到工作者，将分配的任务发送给它们并监督它们的执行。如果执行程序无法将任务分配给指定的工作者，或者任务执行失败，则执行程序会动态地将任务分配给其他工作者节点上的副本。执行程序在处理故障时仅处理失败的查询子树，而不处理整个查询。

Citus有三种基本的执行器类型：实时，路由器和任务跟踪器。它根据每个查询的结构选择动态使用哪个，并且可以在一次查询中使用多个，根据需要为不同的子查询/ `CTEs <https://www.postgresql.org/docs/current/queries-with.html>`_ 分配不同的执行程序以支持SQL功能。这个过程是递归的：如果Citus无法确定如何运行子查询，那么它会检查子子查询。

在高层次上，实时执行器对于处理简单的键值查找和INSERT，UPDATE和DELETE查询非常有用。任务跟踪器更适合较大的SELECT查询，以及路由器执行程序用于访问位于同一位置的单个工作者节点的数据。

可以通过运行PostgreSQL的 `EXPLAIN <https://www.postgresql.org/docs/current/static/sql-explain.html>`_ 命令来显示每个查询的执行程序的选择。这对调试性能问题很有用。

.. _realtime_executor:

实时执行器
~~~~~~~~~~~~~~~

实时执行器是Citus默认使用的执行器。它非常适合快速响应涉及过滤器，聚合和共址联接的查询。实时执行程序为每个分片打开一个与工作者的连接，并将所有查询片段发送给它们。然后，它从每个查询片段中获取结果，合并它们，并将最终结果返回给用户

由于实时执行程序为其发送查询的每个分片维护一个开放连接，因此在处理大量分片时，它可能达到文件描述符/连接限制。在这种情况下，实时执行程序会限制为工作者分配更多任务，以避免因任务太多而压倒他们。通常可以增加现代操作系统上的文件描述符限制以避免限制，并将Citus配置更改为使用实时执行程序。但是，在运行复杂查询时，这可能不是理想的有效资源管理。对于涉及数千个分片或需要大表连接的查询，您可以使用任务跟踪器执行程序。

此外，当实时执行程序检测到简单的INSERT，UPDATE或DELETE查询时，它会将传入的查询分配给具有目标分片的工作者。然后由工作者PostgreSQL服务器处理该查询，并将结果返回给用户。如果对分片副本的修改失败，执行程序将相应的分片副本标记为无效，以保持数据一致性。

.. _router_executor:

路由执行器
~~~~~~~~~~~~~~~

当查询所需的所有数据都存储在单个节点上时，Citus可以将整个查询路由到该节点并在那里运行。然后，通过协调者节点将结果集转发回客户端。路由器执行器负责这种类型的执行。

虽然Citus支持大部分SQL功能，即使是跨节点查询，但路由器执行的优势在于100％的SQL覆盖率。在节点内执行的查询在功能齐全的PostgreSQL工作者实例中运行。路由器执行的缺点是只使用一台计算机执行查询的并行性降低。

任务跟踪执行器
~~~~~~~~~~~~~~~~~~~

任务跟踪器执行程序非常适合长时间运行的复杂数据仓库查询。此执行程序仅为每个工作者打开一个连接，并将所有片段查询分配给工作者上的任务跟踪器守护程序。然后，任务跟踪器守护程序会定期安排新任务并查看完成情况。协调者的执行器定期检查这些任务跟踪器，看看他们的任务是否完成。

工作者上的每个任务跟踪器守护程序也确保同时执行最多citus.max_running_tasks_per_node个任务。当查询不是从内存中提供时，这种并发限制有助于避免磁盘I/O争用。任务跟踪执行程序的设计目的是有效地处理复杂的查询，这些查询需要在工作者之间重新分区和重排中间数据。

.. _push_pull_execution:

子查询/CTE推拉执行
~~~~~~~~~~~~~~~~~~~~~~~

如有必要，Citus可以将子查询和CTE的结果收集到协调者节点中，然后将它们推送回工作者以供外部查询使用。这允许Citus支持更多种类的SQL构造，甚至可以在查询及其子查询之间混合执行程序类型。

例如，在WHERE子句中具有子查询有时不能与主查询同时执行内联，而必须单独完成。假设Web分析应用程序维护一个使用``page_id``分区的表``visits``。要查询访问次数最多的前20个页面上的访问会话的数量，我们可以使用子查询来查找页面列表，然后使用外部查询来计数会话。

.. code-block:: sql

  SELECT page_id, count(distinct session_id)
  FROM visits
  WHERE page_id IN(
    SELECT page_id
    FROM visits
    GROUP BY page_id
    ORDER BY count(*) DESC
    LIMIT 20
  )
  GROUP BY page_id;

实时执行程序希望根据page_id针对每个分片运行此查询的一个片段，计算不同的session_id，并在协调者上组合结果。但是，子查询中的LIMIT意味着子查询不能作为片段的一部分执行。通过递归计划查询，Citus可以单独运行子查询，将结果推送给所有工作者，运行主片段查询，并将结果拉回协调者。“推拉式”设计支持如上所述的子查询。

让我们通过查看此查询的 `EXPLAIN <https://www.postgresql.org/docs/current/static/sql-explain.html>`_ 输出来查看此操作。它相当复杂：

::

  GroupAggregate (cost=0.00..0.00 rows=0 width=0)
    Group Key: remote_scan.page_id
    ->  Sort (cost=0.00..0.00 rows=0 width=0)
      Sort Key: remote_scan.page_id
      ->  Custom Scan(Citus Real-Time) (cost=0.00..0.00 rows=0 width=0)
        ->  Distributed Subplan 6_1
          ->  Limit (cost=0.00..0.00 rows=0 width=0)
            ->  Sort (cost=0.00..0.00 rows=0 width=0)
              Sort Key: COALESCE((pg_catalog.sum((COALESCE((pg_catalog.sum(remote_scan.worker_column_2))::bigint, '0'::bigint))))::bigint, '0'::bigint) DESC
              ->  HashAggregate (cost=0.00..0.00 rows=0 width=0)
                Group Key: remote_scan.page_id
                ->  Custom Scan(Citus Real-Time) (cost=0.00..0.00 rows=0 width=0)
                  Task Count: 32
                  Tasks Shown: One of 32
                  ->  Task
                    Node: host=localhost port=5433 dbname=postgres
                    ->  Limit (cost=1883.00..1883.05 rows=20 width=12)
                      ->  Sort (cost=1883.00..1965.54 rows=33017 width=12)
                        Sort Key:(count(*)) DESC
                        ->  HashAggregate (cost=674.25..1004.42 rows=33017 width=12)
                          Group Key: page_id
                          ->  Seq Scan on visits_102264 visits (cost=0.00..509.17 rows=33017 width=4)
        Task Count: 32
        Tasks Shown: One of 32
        ->  Task
          Node: host=localhost port=5433 dbname=postgres
          ->  HashAggregate (cost=734.53..899.61 rows=16508 width=8)
            Group Key: visits.page_id, visits.session_id
            ->  Hash Join (cost=17.00..651.99 rows=16508 width=8)
              Hash Cond:(visits.page_id = intermediate_result.page_id)
              ->  Seq Scan on visits_102264 visits (cost=0.00..509.17 rows=33017 width=8)
              ->  Hash (cost=14.50..14.50 rows=200 width=4)
                ->  HashAggregate (cost=12.50..14.50 rows=200 width=4)
                  Group Key: intermediate_result.page_id
                  ->  Function Scan on read_intermediate_result intermediate_result (cost=0.00..10.00 rows=1000 width=4)

让我们把它拆开，并检查每一块。

::

  GroupAggregate (cost=0.00..0.00 rows=0 width=0)
    Group Key: remote_scan.page_id
    ->  Sort (cost=0.00..0.00 rows=0 width=0)
      Sort Key: remote_scan.page_id

树的根是协调者对工作者节点的结果所做的。在这种情况下，它将它们分组，GroupAggregate要求首先对它们进行排序。

::

      ->  Custom Scan(Citus Real-Time) (cost=0.00..0.00 rows=0 width=0)
        ->  Distributed Subplan 6_1
  .

自定义扫描有两个大的子树，从“分布式子计划”开始。

::

          ->  Limit (cost=0.00..0.00 rows=0 width=0)
            ->  Sort (cost=0.00..0.00 rows=0 width=0)
              Sort Key: COALESCE((pg_catalog.sum((COALESCE((pg_catalog.sum(remote_scan.worker_column_2))::bigint, '0'::bigint))))::bigint, '0'::bigint) DESC
              ->  HashAggregate (cost=0.00..0.00 rows=0 width=0)
                Group Key: remote_scan.page_id
                ->  Custom Scan(Citus Real-Time) (cost=0.00..0.00 rows=0 width=0)
                  Task Count: 32
                  Tasks Shown: One of 32
                  ->  Task
                    Node: host=localhost port=5433 dbname=postgres
                    ->  Limit (cost=1883.00..1883.05 rows=20 width=12)
                      ->  Sort (cost=1883.00..1965.54 rows=33017 width=12)
                        Sort Key:(count(*)) DESC
                        ->  HashAggregate (cost=674.25..1004.42 rows=33017 width=12)
                          Group Key: page_id
                          ->  Seq Scan on visits_102264 visits (cost=0.00..509.17 rows=33017 width=4)
  .

工作者节点为32个分片中的每一个运行上面的内容（Citus选择一个代表进行显示)。我们可以识别 ``IN(…)`` 子查询的所有部分：排序，分组和限制。当所有工作者完成此查询后，他们将其输出发送回协调者，协调者将其作为“中间结果”放在一起。

::

        Task Count: 32
        Tasks Shown: One of 32
        ->  Task
          Node: host=localhost port=5433 dbname=postgres
          ->  HashAggregate (cost=734.53..899.61 rows=16508 width=8)
            Group Key: visits.page_id, visits.session_id
            ->  Hash Join (cost=17.00..651.99 rows=16508 width=8)
              Hash Cond:(visits.page_id = intermediate_result.page_id)
  .

Citus在第二个子树中开始另一个实时工作。它将计算访问中的不同会话。它使用JOIN连接中间结果。中间结果将帮助它限制在前20页。

::

              ->  Seq Scan on visits_102264 visits (cost=0.00..509.17 rows=33017 width=8)
              ->  Hash (cost=14.50..14.50 rows=200 width=4)
                ->  HashAggregate (cost=12.50..14.50 rows=200 width=4)
                  Group Key: intermediate_result.page_id
                  ->  Function Scan on read_intermediate_result intermediate_result (cost=0.00..10.00 rows=1000 width=4)
  .

工作者使用一个 ``read_intermediate_result`` 函数在内部检索中间结果，该函数从协调者节点复制的文件中加载数据。

此示例显示了Citus如何使用分布式子计划在多个步骤中执行查询，以及如何使用EXPLAIN来了解分布式查询执行。

.. _postgresql_planner_executor:

PostgreSQL规划器和执行器
------------------------------

一旦分布式执行程序将查询片段发送给工作者，它们就像常规的PostgreSQL查询一样处理。该工作者上的PostgreSQL规划器选择最佳的计划，以便在相应的分片表上本地执行该查询。然后，PostgreSQL执行器运行该查询并将查询结果返回给分布式执行器。您可以从PostgreSQL手册中了解有关PostgreSQL `规划器 <http://www.postgresql.org/docs/current/static/planner-optimizer.html>`_ 和 `执行器 <http://www.postgresql.org/docs/current/static/executor.html>`_ 的更多信息。最后，分布式执行器将结果传递给协调者进行最终聚合。
