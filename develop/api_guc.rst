.. _configuration:

配置参考
=======================

有各种配置参数会影响Citus的行为。这些包括标准PostgreSQL参数和Citus特定参数。要了解更多有关PostgreSQL配置参数的信息，可以访问PostgreSQL文档的 `运行时配置 <http://www.postgresql.org/docs/current/static/runtime-config.html>`_ 部分。

本参考的其余部分旨在讨论Citus特定的配置参数。这些参数可以使用类似PostgreSQL参数修改postgresql.conf的方式设置, 或 `使用SET命令 <http://www.postgresql.org/docs/current/static/config-setting.html>`_.

例如，您可以使用以下命令更新设置：
As an example you can update a setting with:

.. code-block:: postgresql

    ALTER DATABASE citus SET citus.multi_task_query_log_level = 'log';


一般配置
---------------------------------------

citus.max_worker_nodes_tracked (integer)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

Citus在协调节点上的分片哈希表中跟踪工作节点的位置及其成员资格。此配置值限制哈希表的大小，因此可以跟踪工作节点的数量。此设置的默认值为2048。此参数只能在服务器启动时设置，并且在协调节点上有效。

citus.use_secondary_nodes (enum)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

设置为SELECT查询选择节点时要使用的策略。
如果将其设置为'always'，则规划器将仅查询在pg_dist_node中节点角色标记为'secondary'的节点。

此枚举支持的值为:

* **never:** (默认) 所有读取都发生在主节点上.

* **always:** 读取针对辅助节点运行，并禁用insert/update语句。

citus.cluster_name (text)
$$$$$$$$$$$$$$$$$$$$$$$$$

通知协调器节点规划器它协调哪个集群。
一旦设置了cluster_name，规划器将仅查询该集群中的工作节点。

.. _enable_version_checks:

citus.enable_version_checks (boolean)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

升级Citus版本需要重新启动服务器(以获取新的共享库)，以及运行ALTER EXTENSION UPDATE命令。执行这两个步骤失败可能会导致错误或崩溃。因此，Citus验证代码的版本和扩展的版本是否匹配，如果不匹配，就会出现错误。

此值默认为true，对协调器有效。在极少数情况下，复杂的升级过程可能需要将此参数设置为false，从而禁用检查。

citus.log_distributed_deadlock_detection (boolean)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

是否在服务器日志中记录分布式死锁检测相关处理。它默认为false。

citus.distributed_deadlock_detection_factor (floating point)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

设置在检查分布式死锁之前等待的时间。特别是等待的时间将是这个值乘以PostgreSQL的 `deadlock_timeout <https://www.postgresql.org/docs/current/static/runtime-config-locks.html>`_ 设置。默认值为 ``2`` 。值-1禁用分布式死锁检测。

.. _node_conninfo:

citus.node_conninfo (text)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

``citus.node_conninfo`` GUC设置非敏感的 `libpq连接参数 <https://www.postgresql.org/docs/current/static/libpq-connect.html#LIBPQ-PARAMKEYWORDS>`_ 用于所有的节点间的连接。

.. code-block:: postgresql

  -- key=value pairs separated by spaces.
  -- For example, ssl options:

  ALTER DATABASE foo
  SET citus.node_conninfo =
    'sslrootcert=/path/to/citus.crt sslmode=verify-full';

Citus仅授予选项的白名单子集，即：

* application_name
* connect_timeout
* gsslib†
* keepalives
* keepalives_count
* keepalives_idle
* keepalives_interval
* krbsrvname†
* sslcompression
* sslcrl
* sslmode  (自Citus 8.1起默认为"require")
* sslrootcert

*(† = 运行时受可选PostgreSQL功能限制)*

.. _override_table_visibility:

citus.override_table_visibility (boolean)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

.. note::

   此GUC仅对Citus MX有影响。

分片以常规表的形式存储在工作节点上，并在其名称后面附加标识符。与常规Citus不同，默认情况下，当用户在psql中运行 ``\d`` 时，Citus MX不会在表列表中显示分片。但是，通过更新GUC，可以在MX中显示分片：

.. code-block:: psql

   SET citus.override_table_visibility TO FALSE;

::

   \d

   +----------+--------------------+--------+----------+
   | Schema   | Name               | Type   | Owner    |
   |----------+--------------------+--------+----------|
   | public   | test_table         | table  | citus    |
   | public   | test_table_102041  | table  | citus    |
   | public   | test_table_102043  | table  | citus    |
   | public   | test_table_102045  | table  | citus    |
   | public   | test_table_102047  | table  | citus    |
   +----------+--------------------+--------+----------+

现在，分片 ``test_table`` (``test_table_<n>``)出现在列表中。

查看分片的另一种方法是查询 :ref:`citus_shards_on_worker <worker_shards>` 视图。

查询统计
---------------------------

citus.stat_statements_purge_interval (integer)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

.. note::

   该GUC是Citus企业版的一部分。请 `联系我们 <https://www.citusdata.com/about/contact_us>`_ 以获取此功能。

设置清除频率, 其维护守护进程从 :ref:`citus_stat_statements <citus_stat_statements>` 中删除在 ``pg_stat_statements`` 中不匹配记录。
此配置值设置清除之间的时间间隔以秒为单位，默认值为10.值为0将禁用清除。

.. code-block:: psql

   SET citus.stat_statements_purge_interval TO 5;

此参数在协调器上有效，可以在运行时更改。

citus.stat_statements_max (integer)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

.. note::

   该GUC是Citus企业版的一部分。请 `联系我们 <https://www.citusdata.com/about/contact_us>`_ 以获取此功能。

要存储在 :ref:`citus_stat_statements <citus_stat_statements>` 中的最大行数。默认为50000，可以更改为1000 - 10000000范围内的任何值。
请注意，每行需要140个字节的存储空间，因此将stat_statements_max设置为最大值10M将占用1.4GB内存。

在重新启动PostgreSQL之前，更改此GUC将不会生效。

数据加载
---------------------------

citus.multi_shard_commit_protocol (enum)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

设置提交协议，以便用于在散列分布式表上执行COPY。在每个单独的分片放置上，COPY在事务块中执行，以确保在COPY期间发生错误时不会摄入任何数据。但是，有一个特殊的失败案例，其中COPY在所有放置上成功，但在所有事务提交之前发生(硬件)故障。通过在以下提交协议之间进行选择，此参数可用于防止数据丢失：

* **2pc:** (default) 在分片放置上执行复制的事务, 首先用PostgreSQL的 `两阶段提交 <http://www.postgresql.org/docs/current/static/sql-prepare-transaction.html>`_ 准备数据, 然后提交。失败的提交可以分别被手动恢复或使用COMMIT PREPARED 或 ROLLBACK PREPARED中止。可以分别使用COMMIT PREPARED或ROLLBACK PREPARED手动恢复或中止失败的提交。在使用2pc时，应在所有工作者上增加 `max_prepared_transactions <http://www.postgresql.org/docs/current/static/runtime-config-resource.html>`_ ，通常与max_connections的值相同。

* **1pc:** 在单轮中提交对分片放置执行COPY的事务。数据可能会丢失, 如果在所有位置COPY成功后提交失败(很少见)。

.. _replication_factor:

citus.shard_replication_factor (integer)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

设置分片的复制因子，也就是将要放置分片的节点数，默认为1。此参数可在运行时设置，对协调器有效。此参数的理想值取决于群集的大小和节点故障率。例如，你可能需要增加此复制因子, 假如您运行大型集群, 并频繁地观察到节点故障。

citus.shard_count (integer)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

设置散列分区表的分片数目，默认为32。
在创建散列分区表时，:ref:`create_distributed_table <create_distributed_table>` UDF使用此值。此参数可以在运行时设置，并对协调者起作用。

citus.shard_max_size (integer)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

设置分片在被分割之前将增长到的最大大小 ，默认为1GB。当一个分片的源文件大小（它将用于分段）超过此配置值时，数据库会确保创建新分片。此参数可以在运行时设置，并对协调者起作用。

.. Comment out this configuration as currently COPY only support random
   placement policy.
.. citus.shard_placement_policy (enum)
   $$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

   Sets the policy to use when choosing nodes for placing newly created shards. When using the \\copy command, the coordinator needs to choose the worker nodes on which it will place the new shards. This configuration value is applicable on the coordinator and specifies the policy to use for selecting these nodes. The supported values for this parameter are :-

   * **round-robin:** The round robin policy is the default and aims to distribute shards evenly across the cluster by selecting nodes in a round-robin fashion. This allows you to copy from any node including the coordinator node.

   * **local-node-first:** The local node first policy places the first replica of the shard on the client node from which the \\copy command is being run. As the coordinator node does not store any data, the policy requires that the command be run from a worker node. As the first replica is always placed locally, it provides better shard placement guarantees.

规划器配置
------------------------------------------------

citus.limit_clause_row_fetch_count (integer)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

设置每个任务要获取的行数, 为limit子句优化。在有些情况下，具有limit子句的select查询可能需要从每个任务获取所有行以生成结果。在更适合使用近似值得情况下，此配置值设置从每个分片中获取的行数。Limit近似值默认情况下是禁用的，此参数设置为-1。此值可以在运行时设置，并且对协调器有效。

citus.count_distinct_error_rate (floating point)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

Citus可以使用postgresql-hll扩展计算count(distinct)近似值。此配置项在计算count(distinct)时设置所需的错误率。0.0，这是默认值，禁用count(distinct)的近似值; 1.0也不能保证结果的准确性。我们建议将此参数设置为0.005以获得最佳效果。此值可以在运行时设置，并且对协调器有效。

citus.task_assignment_policy (enum)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

.. note::

   仅当 :ref:`shard_replication_factor <replication_factor>` 大于1时，或者针对 :ref:`reference_tables` 的查询，此GUC才适用。

设置将任务分配给工作者时使用的策略。协调员根据分片位置为工作者分配任务。此配置值指定进行这些分配时要使用的策略。目前，有三种可能的任务分配策略可以使用。

* **greedy:** 贪婪的策略是默认的，用于在工作者之间平均分配任务。

* **round-robin:** 循环策略以循环方式为工作者分配任务，在不同的副本之间交替。当表的分片数目低于工作者数目时，这可以实现更好的集群利用率。

* **first-replica:** 第一个副本策略根据分片的放置(副本)的插入顺序分配任务。换句话说，分片的片段查询只是分配给具有该分片的第一个副本的工作者。这种方法允许您对哪些分片将在哪些节点上使用(即更强的内存驻留保证)有很强的保证。

此参数可以在运行时设置，并且对协调器有效。

中间数据传输
-------------------------------------------------------------------

citus.binary_worker_copy_format (boolean)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

使用二进制复制格式在工作者之间传输中间数据。在大型表连接期间，Citus可能必须在不同工作者之间动态地重新分配和清洗数据。默认情况下，此数据以文本格式传输。启用此参数指示数据库使用PostgreSQL的二进制序列化格式来传输此数据。此参数对工作者有效，需要在postgresql.conf文件中更改。编辑配置文件后，用户可以发送SIGHUP信号或重新启动服务器以使此更改生效。

citus.binary_master_copy_format (boolean)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

使用二进制复制格式在协调者和工作者之间传输数据。运行分布式查询时，工作者将其中间结果传输到协调者以进行最终聚合。默认情况下，此数据以文本格式传输。启用此参数指示数据库使用PostgreSQL的二进制序列化格式来传输此数据。此参数可以在运行时设置，并且对协调者有效。

citus.max_intermediate_result_size (integer)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

CTE和复杂子查询的中间结果的最大大小（KB）。默认值为1GB，值为-1表示没有限制。超出限制的查询将被取消并生成错误消息。

DDL
-------------------------------------------------------------------

.. _enable_ddl_prop:

citus.enable_ddl_propagation (boolean)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

指定是否自动将DDL更改从协调者传播到所有工作者。默认值是true。由于某些架构更改需要对表进行访问独占锁定，并且因为自动传播按顺序应用于所有工作者，因此可能会使Citus群集暂时响应性降低。您可以选择禁用此设置并手动传播更改。

.. note::

  有关DDL传播支持的列表，请参阅 :ref:`ddl_prop_support`.

执行器配置
------------------------------------------------------------

citus.all_modifications_commutative
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

Citus强制执行交换性规则，并为修改操作获取适当的锁，以确保行为的正确性。例如，它假定 INSERT 语句与另一个 INSERT 语句通信，但不与 UPDATE 或 DELETE 语句通信。同样，它假定 UPDATE 或 DELETE 语句不与另一个 UPDATE 或 DELETE 语句通信。这意味着 UPDATEs 和 DELETEs 要求Citus获得更强的锁。

如果您有UPDATE语句, 它与您的INSERTs或其他UPDATE交替，那么您可以通过将此参数设置为true来放宽这些交换假设。当此参数设置为true时，所有命令都被视为可交换，并声明共享锁，这可以提高整体吞吐量。此参数可以在运行时设置，并且对协调器有效。

citus.max_task_string_size (integer)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

设置工作者任务调用字符串的最大大小(以字节为单位)。更改此值需要重新启动服务器，不能在运行时更改。

活动的工作者的任务在主节点上的共享哈希表中跟踪。此配置值限制单个工作者任务的最大大小，并影响预分配共享内存的大小。

最小值: 8192, 最大值 65536, 默认值 12288


citus.remote_task_check_interval (integer)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

设置频率, Citus用任务跟踪执行器检查管理的作业的状态。默认为10毫秒。协调器将任务分配给工作者，然后定期检查每个任务的进度。此配置值设置两个后续检查之间的时间间隔。此参数在协调者上有效，可以在运行时设置。

citus.task_executor_type (enum)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

Citus有两种执行器类型，用于运行分布式SELECT查询。可以通过设置此配置参数来选择所需的执行器。此参数可接受的值为：

* **real-time:** 实时执行器是默认执行器，当您需要快速响应涉及聚合和跨多个分片的共址连接的查询时，它是最佳的。

* **task-tracker:** 任务跟踪器执行器非常适合长时间运行的复杂查询，这些查询需要跨工作节点进行数据混洗和高效的资源管理。

此参数可以在运行时设置，并且对协调者有效。有关执行程序的更多详细信息，可以访问我们文档的 :ref:`distributed_query_executor` 部分。

.. _multi_task_logging:

citus.multi_task_query_log_level (enum)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

为任何生成多个任务的查询设置日志级别(即，这个查询会击中多个分片)。这在多租户应用程序迁移期间非常有用，因为您可以选择此类查询的错误或警告，以查找它们并向其添加tenant_id过滤器。此参数可以在运行时设置，并且对协调器有效。此参数的默认值为“off”。

此枚举支持的值为：

* **off:** 关闭任何生成多个任务的查询(即跨多个切分)的日志

* **debug:** 记录严重性级别是DEBUG的语句。

* **log:** 记录严重性级别是LOG的语句。日志行将包括运行的SQL查询。

* **notice:** 记录严重性级别是NOTICE的语句。

* **warning:** 记录严重性级别是WARNING的语句。

* **error:** 记录严重性级别是ERROR的语句。

请注意， :code:`error` 在开发测试期间使用它可能很有用，级别较低的日志比如 :code:`log` 在实际生产部署期间使用。选择 ``log`` 将导致多任务查询出现在数据库日志中，查询显示在"STATEMENT."之后。

.. code-block:: text

  LOG:  multi-task query about to be executed
  HINT:  Queries are split to multiple tasks if they have to be split into several queries on the workers.
  STATEMENT:  select * from foo;

实时执行器配置
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

Citus查询规划器首先剪掉与查询无关的分片，然后将计划交给实时执行器。为了执行计划，实时执行器打开一个连接，并为每个未删去的的分片使用两个文件描述符。如果查询命中大量分片，则执行器可能需要打开比 max_connections 更多的连接，或者使用比 max_files_per_process 更多的文件描述符。

在这种情况下，实时执行器将开始限制任务以防止压跨工作者资源。由于此限制可能会降低查询性能，因此实时执行器将发出适当的警告，建议可能需要增加这些参数才能保持所需的性能。下面简要讨论这些参数。

max_connections (integer)
************************************************

设置与数据库服务器的最大并发连接数。默认值通常为100个连接，但如果您的内核设置将不支持它，则可能会更少(在initdb期间确定)。实时执行器为其发送查询的每个分片维护一个打开的连接。增加此配置参数将允许执行器具有更多并发连接，从而并行处理更多分片。这个参数必须在工作者和协调器上进行更改，并且只能在服务器启动期间进行。

max_files_per_process (integer)
*******************************************************

设置每个服务器进程同时打开文件的最大数目，默认为1000。实时执行器需要为它发送查询的每个分片提供两个文件描述符。增加此配置参数将允许执行器具有更多打开的文件描述符，从而并行处理更多分片。必须在工作者和协调者上进行此更改，并且只能在服务器启动期间完成。

.. note::
  除了max_files_per_process之外，还可能需要使用 ulimit 命令增加每个进程的打开文件描述符的内核限制。

citus.enable_repartition_joins (boolean)
****************************************

通常，尝试用实时执行器执行 :ref:`repartition_joins` 将失败并显示错误消息。但是，设置 ``citus.enable_repartition_joins`` 为true, 允许Citus临时切换到任务跟踪器执行器以执行连接。默认值为false。

任务跟踪器执行器配置
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

citus.task_tracker_delay (integer)
**************************************************

设置任务跟踪器在任务管理循环之间的休眠时间，默认为200毫秒。任务跟踪器进程定期唤醒，遍历分配给它的所有任务，并安排和执行这些任务。然后，任务跟踪器在再次遍历这些任务之前休眠一段时间。此配置值确定睡眠时段的长度。此参数对工作者有效，需要在postgresql.conf文件中更改。编辑配置文件后，用户可以发送SIGHUP信号或重启服务器以使更改生效。

此参数可以减少, 可降低由任务跟踪执行器引起的延迟, 通过减少管理循环之间的时间间隔。这在分片查询非常短并因此非常定期更新其状态的情况下非常有用。

citus.max_tracked_tasks_per_node (integer)
****************************************************************

设置每个节点的最大跟踪任务数，默认为1024。这个配置值限制了用于跟踪分配任务的哈希表的大小，由此限制了在任何给定时间可以跟踪的任务的最大数量。这个值只能在服务器启动时设置，并且对工作者有效。

如果希望每个工作节点能够跟踪更多任务，则需要增加此参数。如果此值低于所需值，则Citus会在工作节点上输出错误消息，说明它超出了共享内存，并且还提供了一个提示，指示增加此参数可能有所帮助。

citus.max_assign_task_batch_size (integer)
*******************************************

协调者上的任务跟踪器执行器同步地将任务分批分配给工作者的守护程序。这个参数设置单个批次中要分配的最大任务数。选择更大的批量大小可以更快地分配任务。但是，如果工作者数量很多，那么所有工作者可能需要更长的时间来完成任务。此参数可以在运行时设置，并且对协调者有效。

citus.max_running_tasks_per_node (integer)
****************************************************************

任务跟踪器进程会恰当的调度和执行分配给它的任务。这个配置值设置一个节点上在任何给定时间并发执行的最大任务数，默认为8。此参数在工作节点上有效，需要在postgresql.conf文件中更改。编辑配置文件后，用户可以发送SIGHUP信号或重启服务器以使更改生效。

此配置条目可确保您没有多个任务同时访问磁盘，并有助于避免磁盘I/O争用。如果您的查询是从内存或SSD提供的，则可以不必担心增加max_running_tasks_per_node。

citus.partition_buffer_size (integer)
************************************************

设置用于分区操作的缓冲区大小，默认为8MB。Citus允许在连接两个大表时将表数据重新分区为多个文件。在此分区缓冲区填满后，重新分区的数据将刷新到磁盘上的文件中。此配置项可以在运行时设置，对工作者有效。

解释输出
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

citus.explain_all_tasks (boolean)
************************************************

默认情况下，Citus 在分布式查询上运行 `EXPLAIN <http://www.postgresql.org/docs/current/static/sql-explain.html>`_ 时显示单个任意任务的输出。在大多数情况下，解释输出在不同任务之间是相似的。有时候，一些任务的计划会有所不同，或者执行时间会更长。在这些情况下，启用此参数可能很有用，之后EXPLAIN输出将包含所有任务。这可能会导致EXPLAIN花费更长时间。
