.. _timeseries:

时间序列数据
===============

在时间序列工作负载中，应用程序（例如某些 :ref:`distributing_by_entity_id`）在归档旧信息时查询最新信息。

为了处理这种工作负载，单节点PostgreSQL数据库通常会使用 `表分区 <https://www.postgresql.org/docs/current/static/ddl-partitioning.html>`_ 将时间排序数据的大表分成多个继承表，每个表包含不同的时间范围。

在多个物理表中存储数据可加快数据过期。在一个大的表，删除行会增加找到哪些行被删除的扫描成本, 然后 `清空 <https://www.postgresql.org/docs/10/static/routine-vacuuming.html>`_ 空白的空间。另一方面，丢弃分区是一种独立于数据大小的快速操作。它相当于简单地删除包含数据的磁盘上的文件。

.. image:: ../images/timeseries-delete-vs-drop.png

对表进行分区还会使索引在每个日期范围内变小和变快。对最近数据进行操作的查询可能在适合内存的“热”索引上运行。这加快了读取速度。

.. image:: ../images/timeseries-multiple-indices-select.png

插入也有更小的索引来更新，所以它们也更快。

.. image:: ../images/timeseries-multiple-indices-insert.png

在以下情况下，基于时间的分区最有意义：

1. 大多数查询访问最新数据的一小部分
2. 旧数据定期过期(deleted/dropped)

请记住，在错误的情况下，读取所有这些分区对开销的伤害大于帮助。但是在适当的情况下，这是非常有帮助的。例如，当保存一年的时间序列数据并定期查询最近一周的数据时。

在Citus上缩放时间序列数据
--------------------------------

我们可以将单节点表分区技术与Citus的分布式分片技术相结合，构建一个可伸缩的时间序列数据库。这是两全其美的。它在Postgres 10的声明式表分区上尤其优雅。

.. image:: ../images/timeseries-sharding-and-partitioning.png

例如，让我们分发 *和* 分区一个包含历史 `GitHub事件数据 <https://examples.citusdata.com/events.csv>`_ 的表。

这个GitHub数据集中的每个记录都表示在GitHub中创建的一个事件，以及与该事件相关的关键信息，如事件类型、创建日期和创建该事件的用户。

第一步是按时间创建和分区表，就像我们在单节点PostgreSQL数据库中所做的那样:

.. code-block:: postgresql

  -- the separate schema will be useful later
  CREATE SCHEMA github;

  -- declaratively partitioned table
  CREATE TABLE github.events (
    event_id bigint,
    event_type text,
    event_public boolean,
    repo_id bigint,
    payload jsonb,
    repo jsonb, actor jsonb,
    org jsonb,
    created_at timestamp
  ) PARTITION BY RANGE (created_at);

请注意 ``PARTITION BY RANGE (created_at)``。这告诉Postgres该表将按有序范围中的 ``created_at`` 列进行分区。不过，我们还没有为特定范围创建任何分区。

在创建特定分区之前，让我们在Citus中分布表。我们将用 ``repo_id`` 对其进行分片，这意味着事件将被聚集到每个存储库的分片中。

.. code-block:: postgresql

  SELECT create_distributed_table('github.events', 'repo_id');

此时，Citus已跨工作节点为此表创建了分片。在内部，每个分片都是一个名为 ``github.events_N`` 的表，每个分片标识符为N。此外，Citus传播了分区信息，并且每个分片都已声明 ``Partition key: RANGE (created_at)``。

分区表不能直接包含数据，它更像是跨分区的视图。因此，分片尚未准备好保存数据。我们需要手动创建分区并指定它们的时间范围，之后我们可以插入与范围匹配的数据。

.. code-block:: postgresql

  -- manually make a partition for 2016 events
  CREATE TABLE github.events_2016 PARTITION OF github.events
  FOR VALUES FROM ('2016-01-01') TO ('2016-12-31');

协调者节点现在具有表 ``github.events`` 和 ``github.events_2016``。Citus会将分区创建传播到所有分片，为每个分片创建一个分区。

自动创建分区
-----------------------------

在上一节中，我们手动创建了 ``github.events`` 表的分区。继续这样做是很繁琐的，特别是当使用较少的分区保存不到一年的数据时。让 `pg_partman 扩展 <https://github.com/keithf4/pg_partman>`_ 按需自动创建分区更令人愉快。pg_partman的核心功能与Citus一起开箱即用，当它与本机分区一起使用时。

首先克隆，构建, 并安装pg_partman扩展。然后告诉partman我们想要创建每个持有一小时数据的分区。这将创建初始空的每小时分区：

.. code-block:: sql

  CREATE SCHEMA partman;
  CREATE EXTENSION pg_partman WITH SCHEMA partman;

  -- Partition the table into hourly ranges of "created_at"
  SELECT partman.create_parent('github.events', 'created_at', 'native', 'hourly');
  UPDATE partman.part_config SET infinite_time_partitions = true;

运行 ``\d+ github.events`` 现在会显示更多的分区:

::

  \d+ github.events
                                                  Table "github.events"
      Column    |            Type             | Collation | Nullable | Default | Storage  | Stats target | Description
  --------------+-----------------------------+-----------+----------+---------+----------+--------------+-------------
   event_id     | bigint                      |           |          |         | plain    |              |
   event_type   | text                        |           |          |         | extended |              |
   event_public | boolean                     |           |          |         | plain    |              |
   repo_id      | bigint                      |           |          |         | plain    |              |
   payload      | jsonb                       |           |          |         | extended |              |
   repo         | jsonb                       |           |          |         | extended |              |
   actor        | jsonb                       |           |          |         | extended |              |
   org          | jsonb                       |           |          |         | extended |              |
   created_at   | timestamp without time zone |           |          |         | plain    |              |
  Partition key: RANGE (created_at)
  Partitions: github.events_p2018_01_15_0700 FOR VALUES FROM ('2018-01-15 07:00:00') TO ('2018-01-15 08:00:00'),
              github.events_p2018_01_15_0800 FOR VALUES FROM ('2018-01-15 08:00:00') TO ('2018-01-15 09:00:00'),
              github.events_p2018_01_15_0900 FOR VALUES FROM ('2018-01-15 09:00:00') TO ('2018-01-15 10:00:00'),
              github.events_p2018_01_15_1000 FOR VALUES FROM ('2018-01-15 10:00:00') TO ('2018-01-15 11:00:00'),
              github.events_p2018_01_15_1100 FOR VALUES FROM ('2018-01-15 11:00:00') TO ('2018-01-15 12:00:00'),
              github.events_p2018_01_15_1200 FOR VALUES FROM ('2018-01-15 12:00:00') TO ('2018-01-15 13:00:00'),
              github.events_p2018_01_15_1300 FOR VALUES FROM ('2018-01-15 13:00:00') TO ('2018-01-15 14:00:00'),
              github.events_p2018_01_15_1400 FOR VALUES FROM ('2018-01-15 14:00:00') TO ('2018-01-15 15:00:00'),
              github.events_p2018_01_15_1500 FOR VALUES FROM ('2018-01-15 15:00:00') TO ('2018-01-15 16:00:00')


默认情况下 ``create_parent`` 在过去创建四个分区，将来创建四个分区，现在创建一个分区，所有这些都基于系统时间。如果需要填充旧数据，可以在调用 ``create_parent`` 或 ``p_premake`` 时指定 ``p_start_partition`` 参数，以便为将来创建分区。有关详细信息，请参阅  `pg_partman 文档 <https://github.com/keithf4/pg_partman/blob/master/doc/pg_partman.md>`_。

随着时间的推移，pg_partman将需要进行一些维护以创建新分区并删除旧分区。无论何时您想触发维护，调用：

.. code-block:: postgresql

  -- disabling analyze is recommended for native partitioning
  -- due to aggressive locks
  SELECT partman.run_maintenance(p_analyze := false);

最好设置定时任务来运行维护功能。可以构建Pg_partman以支持后台工作者（BGW）进程来执行此操作。或者我们可以使用另一个扩展，如 `pg_cron <https://github.com/citusdata/pg_cron>`_：

.. code-block:: postgresql

  SELECT cron.schedule('@hourly', $$
    SELECT partman.run_maintenance(p_analyze := false);
  $$);

一旦建立了定期维护，您就不再需要考虑分区，它们会正常的工作。

最后，要配置pg_partman以删除旧分区，您可以更新 ``partman.part_config`` 表：

.. code-block:: postgresql

  UPDATE partman.part_config
     SET retention_keep_table = false,
         retention = '1 month'
   WHERE parent_table = 'github.events';

现在，只要维护运行，就会自动删除超过一个月的分区。

.. note::

  请注意，Postgres中的本地分区仍然很新，并且有一些怪癖。例如，不能直接在分区表上创建in索引。相反，pg_partman允许您创建一个模板表来为新分区定义索引。分区表上的维护操作还将获得入侵锁，这可能会暂时阻止查询。目前，postgres社区正在进行大量工作来解决这些问题，所以可以预期，postgres中的时间分配只会变得更好。
