.. _metadata_tables:

Citus表和视图
======================

协调者元数据
--------------------

Citus根据分布列将每个分布式表划分为多个逻辑分片。
然后, 协调器维护元数据表，以跟踪统计信息和关于这些分片的健康状况和位置的信息。
在本节中，我们将描述每个元数据表及其架构。登录到协调器节点后，您可以使用SQL查看和查询这些表。

.. _partition_table:

分区表
~~~~~~~~~~~~~~~~~

pg_dist_partition表存储有关数据库中哪些表的分布的元数据。对于每个分布式表，它还存储有关分布方法的信息和有关分布列的详细信息。

+----------------+----------------------+---------------------------------------------------------------------------+
|      Name      |         Type         |       Description                                                         |
+================+======================+===========================================================================+
| logicalrelid   |         regclass     | | 此行对应的分布式表。该值引用pg_class系统目录表中的relfilenode列。       |
|                |                      | |                                                                         |
+----------------+----------------------+---------------------------------------------------------------------------+
|  partmethod    |         char         | | 用于分区/分发的方法。这个的价值对应不同分配方法的列是                   |
|                |                      | | append: 'a'                                                             |
|                |                      | | hash: 'h'                                                               |
|                |                      | | reference table: 'n'                                                    |
+----------------+----------------------+---------------------------------------------------------------------------+
|   partkey      |         text         | | 有关分发列的详细信息，包括列数量，类型和其他相关信息。                  |
+----------------+----------------------+---------------------------------------------------------------------------+
|   colocationid |         integer      | | 此表所属的共址组。同一组中的表允许共存的连接和分布式汇总等优化。        |
|                |                      | | 该值引用了pg_dist_colocation表中的colocationid列。                      |
+----------------+----------------------+---------------------------------------------------------------------------+
|   repmodel     |         char         | | 用于数据复制的方法。此列的值对应不同的复制方法有：                      |
|                |                      | | * 基于声明的复制: 'c'                                                   |
|                |                      | | * postgresql流复制:  's'                                                |
|                |                      | | * 两阶段提交（参考表）: 't'                                             |
+----------------+----------------------+---------------------------------------------------------------------------+

::

    SELECT * from pg_dist_partition;
     logicalrelid  | partmethod |                                                        partkey                                                         | colocationid | repmodel 
    ---------------+------------+------------------------------------------------------------------------------------------------------------------------+--------------+----------
     github_events | h          | {VAR :varno 1 :varattno 4 :vartype 20 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnoold 1 :varoattno 4 :location -1} |            2 | c
     (1 row)

.. _pg_dist_shard:

分片表
~~~~~~~~~~~~~~~~~

pg_dist_shard表存储有关表的各个分片的元数据。这包括有关该分片所属的分布式表的信息以及该分片的分步列的统计信息。对于追加分布式表，这些统计信息对应于分步列的最小值/最大值。在散列分布式表的情况下，它们是分配给该分片的散列令牌范围。这些统计信息用于在SELECT查询期间修剪不相关的分片。

+----------------+----------------------+---------------------------------------------------------------------------+
|      Name      |         Type         |       Description                                                         |
+================+======================+===========================================================================+
| logicalrelid   |         regclass     | | 此分片所属的分布式表。该值引用了pg_class系统目录表中的relfilenode列。   |
+----------------+----------------------+---------------------------------------------------------------------------+
|    shardid     |         bigint       | | 分配给此分片的全局唯一标识符。                                          |
+----------------+----------------------+---------------------------------------------------------------------------+
| shardstorage   |            char      | | 用于此分片的存储类型。不同的存储类型是在下表中讨论。                    |
+----------------+----------------------+---------------------------------------------------------------------------+
| shardminvalue  |            text      | | 对于附加分布式表，分布列的最小值在这个分片中（包括）                    |
|                |                      | | 对于散列分布式表，分配给它的最小散列令牌值分片（包括）。                |
+----------------+----------------------+---------------------------------------------------------------------------+
| shardmaxvalue  |            text      | | 对于附加分布式表，分发列的最大值在这个分片（包括）中。                  |
|                |                      | | 对于散列分布式表，分配给它的最大散列令牌值分片（包括）。                |
+----------------+----------------------+---------------------------------------------------------------------------+

::

    SELECT * from pg_dist_shard;
     logicalrelid  | shardid | shardstorage | shardminvalue | shardmaxvalue 
    ---------------+---------+--------------+---------------+---------------
     github_events |  102026 | t            | 268435456     | 402653183
     github_events |  102027 | t            | 402653184     | 536870911
     github_events |  102028 | t            | 536870912     | 671088639
     github_events |  102029 | t            | 671088640     | 805306367
     (4 rows)


分片存储类型
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

pg_dist_shard中的shardstorage列指示用于分片的存储类型。下面简要概述了不同的分片存储类型及其表示形式。

+----------------+----------------------+---------------------------------------------------------------------------+
|  存储类型      |  Shardstorage值      |  描述                                                                     |
+================+======================+===========================================================================+
|   TABLE        |           't'        | | 表示分片存储属于常规的数据布式表。                                      |
+----------------+----------------------+---------------------------------------------------------------------------+
|  COLUMNAR      |            'c'       | | 表示分片存储列数据。（用于分布式cstore_fdw表）                          |
+----------------+----------------------+---------------------------------------------------------------------------+
|   FOREIGN      |            'f'       | | 表示分片存储外部数据。（用于分布式file_fdw表）                          |
|+---------------+----------------------+---------------------------------------------------------------------------+


.. _placements:

Shard placement table
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The pg_dist_placement table tracks the location of shard replicas on worker nodes. Each replica of a shard assigned to a specific node is called a shard placement. This table stores information about the health and location of each shard placement.

+----------------+----------------------+---------------------------------------------------------------------------+
|      Name      |         Type         |       Description                                                         |
+================+======================+===========================================================================+
| shardid        |       bigint         | | Shard identifier associated with this placement. This value references  |
|                |                      | | the shardid column in the pg_dist_shard catalog table.                  |
+----------------+----------------------+---------------------------------------------------------------------------+ 
| shardstate     |         int          | | Describes the state of this placement. Different shard states are       |
|                |                      | | discussed in the section below.                                         |
+----------------+----------------------+---------------------------------------------------------------------------+
| shardlength    |       bigint         | | For append distributed tables, the size of the shard placement on the   |
|                |                      | | worker node in bytes.                                                   |
|                |                      | | For hash distributed tables, zero.                                      |
+----------------+----------------------+---------------------------------------------------------------------------+
| placementid    |       bigint         | | Unique auto-generated identifier for each individual placement.         |
+----------------+----------------------+---------------------------------------------------------------------------+
| groupid        |         int          | | Identifier used to denote a group of one primary server and zero or more|
|                |                      | | secondary servers, when the streaming replication model is used.        |
+----------------+----------------------+---------------------------------------------------------------------------+

::

  SELECT * from pg_dist_placement;
    shardid | shardstate | shardlength | placementid | groupid
   ---------+------------+-------------+-------------+---------
     102008 |          1 |           0 |           1 |       1
     102008 |          1 |           0 |           2 |       2
     102009 |          1 |           0 |           3 |       2
     102009 |          1 |           0 |           4 |       3
     102010 |          1 |           0 |           5 |       3
     102010 |          1 |           0 |           6 |       4
     102011 |          1 |           0 |           7 |       4

.. note::

  As of Citus 7.0 the analogous table :code:`pg_dist_shard_placement` has been deprecated. It included the node name and port for each placement:

  ::

    SELECT * from pg_dist_shard_placement;
      shardid | shardstate | shardlength | nodename  | nodeport | placementid 
     ---------+------------+-------------+-----------+----------+-------------
       102008 |          1 |           0 | localhost |    12345 |           1
       102008 |          1 |           0 | localhost |    12346 |           2
       102009 |          1 |           0 | localhost |    12346 |           3
       102009 |          1 |           0 | localhost |    12347 |           4
       102010 |          1 |           0 | localhost |    12347 |           5
       102010 |          1 |           0 | localhost |    12345 |           6
       102011 |          1 |           0 | localhost |    12345 |           7

  That information is now available by joining pg_dist_placement with :ref:`pg_dist_node <pg_dist_node>` on the groupid. For compatibility Citus still provides pg_dist_shard_placement as a view. However we recommend using the new, more normalized, tables when possible.


Shard Placement States
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

Citus manages shard health on a per-placement basis and automatically marks a placement as unavailable if leaving the placement in service would put the cluster in an inconsistent state. The shardstate column in the pg_dist_placement table is used to store the state of shard placements. A brief overview of different shard placement states and their representation is below.


+----------------+----------------------+---------------------------------------------------------------------------+
|  State name    |  Shardstate value    |       Description                                                         |
+================+======================+===========================================================================+
|   FINALIZED    |           1          | | This is the state new shards are created in. Shard placements           |
|                |                      | | in this state are considered up-to-date and are used in query   	       |
|                |                      | | planning and execution.                                                 |
+----------------+----------------------+---------------------------------------------------------------------------+   
|  INACTIVE      |            3         | | Shard placements in this state are considered inactive due to           |
|                |                      | | being out-of-sync with other replicas of the same shard. This           |
|                |                      | | can occur when an append, modification (INSERT, UPDATE or               |
|                |                      | | DELETE ) or a DDL operation fails for this placement. The query         |
|                |                      | | planner will ignore placements in this state during planning and        |
|                |                      | | execution. Users can synchronize the data in these shards with          |
|                |                      | | a finalized replica as a background activity.                           |
+----------------+----------------------+---------------------------------------------------------------------------+
|   TO_DELETE    |            4         | | If Citus attempts to drop a shard placement in response to a            |
|                |                      | | master_apply_delete_command call and fails, the placement is            |
|                |                      | | moved to this state. Users can then delete these shards as a            |
|                |                      | | subsequent background activity.                                         |
+----------------+----------------------+---------------------------------------------------------------------------+


.. _pg_dist_node:

Worker node table
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The pg_dist_node table contains information about the worker nodes in the cluster. 

+----------------+----------------------+---------------------------------------------------------------------------+
|      Name      |         Type         |       Description                                                         |
+================+======================+===========================================================================+
| nodeid         |         int          | | Auto-generated identifier for an individual node.                       |
+----------------+----------------------+---------------------------------------------------------------------------+
| groupid        |         int          | | Identifier used to denote a group of one primary server and zero or more|
|                |                      | | secondary servers, when the streaming replication model is used. By     |
|                |                      | | default it is the same as the nodeid.                                   | 
+----------------+----------------------+---------------------------------------------------------------------------+
| nodename       |         text         | | Host Name or IP Address of the PostgreSQL worker node.                  |
+----------------+----------------------+---------------------------------------------------------------------------+
| nodeport       |         int          | | Port number on which the PostgreSQL worker node is listening.           |
+----------------+----------------------+---------------------------------------------------------------------------+
| noderack       |        text          | | (Optional) Rack placement information for the worker node.              |
+----------------+----------------------+---------------------------------------------------------------------------+
| hasmetadata    |        boolean       | | Reserved for internal use.                                              |
+----------------+----------------------+---------------------------------------------------------------------------+
| isactive       |        boolean       | | Whether the node is active accepting shard placements.                  |
+----------------+----------------------+---------------------------------------------------------------------------+
| noderole       |        text          | | Whether the node is a primary or secondary                              |
+----------------+----------------------+---------------------------------------------------------------------------+
| nodecluster    |        text          | | The name of the cluster containing this node                            |
+----------------+----------------------+---------------------------------------------------------------------------+

::

    SELECT * from pg_dist_node;
     nodeid | groupid | nodename  | nodeport | noderack | hasmetadata | isactive | noderole | nodecluster
    --------+---------+-----------+----------+----------+-------------+----------+----------+ -------------
          1 |       1 | localhost |    12345 | default  | f           | t        | primary  | default
          2 |       2 | localhost |    12346 | default  | f           | t        | primary  | default
          3 |       3 | localhost |    12347 | default  | f           | t        | primary  | default
    (3 rows)

.. _colocation_group_table:

Co-location group table
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The pg_dist_colocation table contains information about which tables' shards should be placed together, or :ref:`co-located <colocation>`. When two tables are in the same co-location group, Citus ensures shards with the same partition values will be placed on the same worker nodes. This enables join optimizations, certain distributed rollups, and foreign key support. Shard co-location is inferred when the shard counts, replication factors, and partition column types all match between two tables; however, a custom co-location group may be specified when creating a distributed table, if so desired.

+------------------------+----------------------+---------------------------------------------------------------------------+
|      Name              |         Type         |       Description                                                         |
+========================+======================+===========================================================================+
| colocationid           |         int          | | Unique identifier for the co-location group this row corresponds to.    |
+------------------------+----------------------+---------------------------------------------------------------------------+
| shardcount             |         int          | | Shard count for all tables in this co-location group                    |
+------------------------+----------------------+---------------------------------------------------------------------------+
| replicationfactor      |         int          | | Replication factor for all tables in this co-location group.            |
+------------------------+----------------------+---------------------------------------------------------------------------+
| distributioncolumntype |         oid          | | The type of the distribution column for all tables in this              |
|                        |                      | | co-location group.                                                      |
+------------------------+----------------------+---------------------------------------------------------------------------+

::

    SELECT * from pg_dist_colocation;
      colocationid | shardcount | replicationfactor | distributioncolumntype 
     --------------+------------+-------------------+------------------------
                 2 |         32 |                 2 |                     20
      (1 row)

.. _citus_stat_statements:

Query statistics table
~~~~~~~~~~~~~~~~~~~~~~

.. note::

  The citus_stat_statements view is a part of Citus Enterprise. Please `contact us <https://www.citusdata.com/about/contact_us>`_ to obtain this functionality.

Citus provides ``citus_stat_statements`` for stats about how queries are being executed, and for whom. It's analogous to (and can be joined with) the `pg_stat_statements <https://www.postgresql.org/docs/current/static/pgstatstatements.html>`_ view in PostgreSQL which tracks statistics about query speed.

This view can trace queries to originating tenants in a multi-tenant application, which helps for deciding when to do :ref:`tenant_isolation`.

+----------------+--------+---------------------------------------------------------+
| Name           | Type   | Description                                             |
+================+========+=========================================================+
| queryid        | bigint | identifier (good for pg_stat_statements joins)          |
+----------------+--------+---------------------------------------------------------+
| userid         | oid    | user who ran the query                                  |
+----------------+--------+---------------------------------------------------------+
| dbid           | oid    | database instance of coordinator                        |
+----------------+--------+---------------------------------------------------------+
| query          | text   | anonymized query string                                 |
+----------------+--------+---------------------------------------------------------+
| executor       | text   | Citus :ref:`executor <distributed_query_executor>` used:|
|                |        | real-time, task-tracker, router, or insert-select       |
+----------------+--------+---------------------------------------------------------+
| partition_key  | text   | value of distribution column in router-executed queries,|
|                |        | else NULL                                               |
+----------------+--------+---------------------------------------------------------+
| calls          | bigint | number of times the query was run                       |
+----------------+--------+---------------------------------------------------------+

.. code-block:: sql

  -- create and populate distributed table
  create table foo ( id int );
  select create_distributed_table('foo', 'id');
  insert into foo select generate_series(1,100);

  -- enable stats
  -- pg_stat_statements must be in shared_preload libraries
  create extension pg_stat_statements;

  select count(*) from foo;
  select * from foo where id = 42;

  select * from citus_stat_statements;

Results:

::

  ┌────────────┬────────┬───────┬───────────────────────────────────────────────┬───────────────┬───────────────┬───────┐
  │  queryid   │ userid │ dbid  │                     query                     │   executor    │ partition_key │ calls │
  ├────────────┼────────┼───────┼───────────────────────────────────────────────┼───────────────┼───────────────┼───────┤
  │ 1496051219 │  16384 │ 16385 │ select count(*) from foo;                     │ real-time     │ NULL          │     1 │
  │ 2530480378 │  16384 │ 16385 │ select * from foo where id = $1               │ router        │ 42            │     1 │
  │ 3233520930 │  16384 │ 16385 │ insert into foo select generate_series($1,$2) │ insert-select │ NULL          │     1 │
  └────────────┴────────┴───────┴───────────────────────────────────────────────┴───────────────┴───────────────┴───────┘

Caveats:

* The stats data is not replicated, and won't survive database crashes or failover
* It's a coordinator node feature, with no :ref:`Citus MX <mx>` support
* Tracks a limited number of queries, set by the ``pg_stat_statements.max`` GUC (default 5000)
* To truncate the table, use the ``citus_stat_statements_reset()`` function

Distributed Query Activity
~~~~~~~~~~~~~~~~~~~~~~~~~~

With :ref:`mx` users can execute distributed queries from any node. Examining the standard Postgres `pg_stat_activity <https://www.postgresql.org/docs/current/static/monitoring-stats.html#PG-STAT-ACTIVITY-VIEW>`_ view on the coordinator won't include those worker-initiated queries. Also queries might get blocked on row-level locks on one of the shards on a worker node. If that happens then those queries would not show up in `pg_locks <https://www.postgresql.org/docs/current/static/view-pg-locks.html>`_ on the Citus coordinator node.

Citus provides special views to watch queries and locks throughout the cluster, including shard-specific queries used internally to build results for distributed queries.

* **citus_dist_stat_activity**: shows the distributed queries that are executing on all nodes. A superset of ``pg_stat_activity``, usable wherever the latter is.
* **citus_worker_stat_activity**: shows queries on workers, including fragment queries against individual shards.
* **citus_lock_waits**: Blocked queries throughout the cluster.

The first two views include all columns of `pg_stat_activity <https://www.postgresql.org/docs/current/static/monitoring-stats.html#PG-STAT-ACTIVITY-VIEW>`_ plus the host host/port of the worker that initiated the query and the host/port of the coordinator node of the cluster.

For example, consider counting the rows in a distributed table:

.. code-block:: postgres

   -- run from worker on localhost:9701

   SELECT count(*) FROM users_table;

We can see the query appear in ``citus_dist_stat_activity``:

.. code-block:: postgres

   SELECT * FROM citus_dist_stat_activity;

   -[ RECORD 1 ]----------+----------------------------------
   query_hostname         | localhost
   query_hostport         | 9701
   master_query_host_name | localhost
   master_query_host_port | 9701
   transaction_number     | 1
   transaction_stamp      | 2018-10-05 13:27:20.691907+03
   datid                  | 12630
   datname                | postgres
   pid                    | 23723
   usesysid               | 10
   usename                | citus
   application_name       | psql
   client_addr            | 
   client_hostname        | 
   client_port            | -1
   backend_start          | 2018-10-05 13:27:14.419905+03
   xact_start             | 2018-10-05 13:27:16.362887+03
   query_start            | 2018-10-05 13:27:20.682452+03
   state_change           | 2018-10-05 13:27:20.896546+03
   wait_event_type        | Client
   wait_event             | ClientRead
   state                  | idle in transaction
   backend_xid            | 
   backend_xmin           | 
   query                  | SELECT count(*) FROM users_table;
   backend_type           | client backend

This query requires information from all shards. Some of the information is in shard ``users_table_102038`` which happens to be stored in localhost:9700. We can see a query accessing the shard by looking at the ``citus_worker_stat_activity`` view:

.. code-block:: postgres

   SELECT * FROM citus_worker_stat_activity;

   -[ RECORD 1 ]----------+-----------------------------------------------------------------------------------------
   query_hostname         | localhost
   query_hostport         | 9700
   master_query_host_name | localhost
   master_query_host_port | 9701
   transaction_number     | 1
   transaction_stamp      | 2018-10-05 13:27:20.691907+03
   datid                  | 12630
   datname                | postgres
   pid                    | 23781
   usesysid               | 10
   usename                | citus
   application_name       | citus
   client_addr            | ::1
   client_hostname        | 
   client_port            | 51773
   backend_start          | 2018-10-05 13:27:20.75839+03
   xact_start             | 2018-10-05 13:27:20.84112+03
   query_start            | 2018-10-05 13:27:20.867446+03
   state_change           | 2018-10-05 13:27:20.869889+03
   wait_event_type        | Client
   wait_event             | ClientRead
   state                  | idle in transaction
   backend_xid            | 
   backend_xmin           | 
   query                  | COPY (SELECT count(*) AS count FROM users_table_102038 users_table WHERE true) TO STDOUT
   backend_type           | client backend

The ``query`` field shows data being copied out of the shard to be counted.

.. note::

  If a router query (e.g. single-tenant in a multi-tenant application, ``SELECT * FROM table WHERE tenant_id = X``) is executed without a transaction block, then master_query_host_name and master_query_host_port columns will be NULL in citus_worker_stat_activity.

To see how ``citus_lock_waits`` works, we can generate a locking situation manually. First we'll set up a test table from the coordinator:

.. code-block:: postgres

   CREATE TABLE numbers AS
     SELECT i, 0 AS j FROM generate_series(1,10) AS i;
   SELECT create_distributed_table('numbers', 'i');

Then, using two sessions on the coordinator, we can run this sequence of statements:

.. code-block:: postgres

   -- session 1                           -- session 2
   -------------------------------------  -------------------------------------
   BEGIN;
   UPDATE numbers SET j = 2 WHERE i = 1;
                                          BEGIN;
                                          UPDATE numbers SET j = 3 WHERE i = 1;
                                          -- (this blocks)

The ``citus_lock_waits`` view shows the situation.

.. code-block:: postgres

   SELECT * FROM citus_lock_waits;

   -[ RECORD 1 ]-------------------------+----------------------------------------
   waiting_pid                           | 88624
   blocking_pid                          | 88615
   blocked_statement                     | UPDATE numbers SET j = 3 WHERE i = 1;
   current_statement_in_blocking_process | UPDATE numbers SET j = 2 WHERE i = 1;
   waiting_node_id                       | 0
   blocking_node_id                      | 0
   waiting_node_name                     | coordinator_host
   blocking_node_name                    | coordinator_host
   waiting_node_port                     | 5432
   blocking_node_port                    | 5432

In this example the queries originated on the coordinator, but the view can also list locks between queries originating on workers (executed with Citus MX for instance).

Tables on all Nodes
-------------------

Citus has other informational tables and views which are accessible on all nodes, not just the coordinator.

.. _pg_dist_authinfo:

Connection Credentials Table
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

  This table is a part of Citus Enterprise Edition. Please `contact us <https://www.citusdata.com/about/contact_us>`_ to obtain this functionality.

The ``pg_dist_authinfo`` table holds authentication parameters used by Citus nodes to connect to one another.

+----------+---------+-------------------------------------------------+
| Name     | Type    | Description                                     |
+==========+=========+=================================================+
| nodeid   | integer | Node id from :ref:`pg_dist_node`, or 0, or -1   |
+----------+---------+-------------------------------------------------+
| rolename | name    | Postgres role                                   |
+----------+---------+-------------------------------------------------+
| authinfo | text    | Space-separated libpq connection parameters     |
+----------+---------+-------------------------------------------------+

Upon beginning a connection, a node consults the table to see whether a row with the destination ``nodeid`` and desired ``rolename`` exists. If so, the node includes the corresponding ``authinfo`` string in its libpq connection. A common example is to store a password, like ``'password=abc123'``, but you can review the `full list <https://www.postgresql.org/docs/current/static/libpq-connect.html#LIBPQ-PARAMKEYWORDS>`_ of possibilities.

The parameters in ``authinfo`` are space-separated, in the form ``key=val``. To write an empty value, or a value containing spaces, surround it with single quotes, e.g., ``keyword='a value'``. Single quotes and backslashes within the value must be escaped with a backslash, i.e., ``\'`` and ``\\``.

The ``nodeid`` column can also take the special values 0 and -1, which mean *all nodes* or *loopback connections*, respectively. If, for a given node, both specific and all-node rules exist, the specific rule has precedence.

::

    SELECT * FROM pg_dist_authinfo;

     nodeid | rolename | authinfo
    --------+----------+-----------------
        123 | jdoe     | password=abc123
    (1 row)

Connection Pooling Credentials
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

  This table is a part of Citus Enterprise Edition. Please `contact us <https://www.citusdata.com/about/contact_us>`_ to obtain this functionality.

If you want to use a connection pooler to connect to a node, you can specify the pooler options using ``pg_dist_poolinfo``. This metadata table holds the host, port and database name for Citus to use when connecting to a node through a pooler.

If pool information is present, Citus will try to use these values instead of setting up a direct connection. The pg_dist_poolinfo information in this case supersedes :ref:`pg_dist_node <pg_dist_node>`.

+----------+---------+---------------------------------------------------+
| Name     | Type    | Description                                       |
+==========+=========+===================================================+
| nodeid   | integer | Node id from :ref:`pg_dist_node`                  |
+----------+---------+---------------------------------------------------+
| poolinfo | text    | Space-separated parameters: host, port, or dbname |
+----------+---------+---------------------------------------------------+

.. note::

   In some situations Citus ignores the settings in pg_dist_poolinfo. For instance :ref:`Shard rebalancing <shard_rebalancing>` is not compatible with connection poolers such as pgbouncer. In these scenarios Citus will use a direct connection.

.. code-block:: sql

   -- how to connect to node 1 (as identified in pg_dist_node)

   INSERT INTO pg_dist_poolinfo (nodeid, poolinfo)
        VALUES (1, 'host=127.0.0.1 port=5433');

.. _worker_shards:

Shards and Indices on MX Workers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

   The citus_shards_on_worker and citus_shard_indexes_on_worker views are relevant in Citus MX only. In the non-MX scenario they contain no rows.

Worker nodes store shards as tables that are ordinarily hidden in Citus MX (see :ref:`override_table_visibility`). The easiest way to obtain information about the shards on each worker is to consult that worker's ``citus_shards_on_worker`` view. For instance, here are some shards on a worker for the distributed table ``test_table``:

.. code-block:: postgres

   SELECT * FROM citus_shards_on_worker ORDER BY 2;
    Schema |        Name        | Type  | Owner
   --------+--------------------+-------+-------
    public | test_table_1130000 | table | citus
    public | test_table_1130002 | table | citus

Indices for shards are also hidden, but discoverable through another view, ``citus_shard_indexes_on_worker``:

.. code-block:: postgres

   SELECT * FROM citus_shard_indexes_on_worker ORDER BY 2;
    Schema |        Name        | Type  | Owner |       Table
   --------+--------------------+-------+-------+--------------------
    public | test_index_1130000 | index | citus | test_table_1130000
    public | test_index_1130002 | index | citus | test_table_1130002

