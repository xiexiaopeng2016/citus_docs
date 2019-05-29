.. _user_defined_functions:

Citus实用功能
=============

本节包含Citus提供的用户定义函数的参考信息。这些函数有助于为Citus提供除标准SQL命令之外的其他分布式功能。

表和分片 DDL
------------
.. _create_distributed_table:

create_distributed_table
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

create_distributed_table() 函数用于定义分布式表，如果它是散列分布式表，则为它创建切分。此函数接受表名，分布列和可选的分布方法，并插入适当的元数据以将表标记为分布式。如果未指定分发方法，则函数默认为'hash'分布。如果表是散列分布式的，则该函数还会根据分片计数和分片复制因子配置值创建工作分片。如果表包含任何行，则它们会自动分发到工作节点。

这个函数替换了master_create_distributed_table()后面跟着master_create_worker_shards()的用法。

参数
************************

**table_name:** 需要分发的表的名称。

**distribution_column:** 要分布该表的列。

**distribution_type:** (可选) 分布表的方法。允许的值是append或hash，默认值为'hash'。

**colocate_with:** (可选) 将当前表包含在另一个表的共存位置组中。
默认情况下，如果表由相同类型的列分布，具有相同的分片计数，并且具有相同的复制因子，那么它们将位于同一位置。
:code:`colocate_with`可能的值是:code:`default`, :code:`none`. 要启动新的协同定位组，还是要与该表共同定位的另一个表的名称。（参见共同定位表。）
code: ' colocate_with '的可能值是:code: ' default ', :code: ' none '来启动一个新的共定位组，或者另一个表的名称来与该表共定位。(参见:ref:`colocation_groups`.)

请记住``colocate_with``的默认值是隐式共址。正如:ref:`colocation`所解释的那样，当表被关联或将被连接时，这可能是一件很棒的事情。但是，当两个表不相关但碰巧对其分发列使用相同的数据类型时，意外地共同定位它们会降低分片重新平衡期间的性能。然而，当两个表不相关，但碰巧对它们的分布列使用相同的数据类型时，意外地将它们放在同一个位置会降低:ref:`shard rebalancing <shard_rebalancing>`期间的性能。表分片将不必要地以“级联”方式移动到一起。

如果新的分布式表与其他表无关，则最好指定``colocate_with => 'none'``。

返回值
********************************

N/A

示例
*************************

此示例通知数据库github_events表应通过repo_id列进行hash分布。

.. code-block:: postgresql

  SELECT create_distributed_table('github_events', 'repo_id');

  -- alternatively, to be more explicit:
  SELECT create_distributed_table('github_events', 'repo_id',
                                  colocate_with => 'github_repo');

有关更多示例，请参阅:ref:`ddl`。

.. _create_reference_table:

create_reference_table
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

create_reference_table()函数用于定义小型引用或维度表。此函数接受表名，并创建仅包含一个分片的分布式表，并复制到每个工作节点。

参数
************************

**table_name:** 需要分发的小维度或引用表的名称。


返回值
********************************

N/A

示例
*************************
这个示例通知数据库，应该将nation表定义为引用表

.. code-block:: postgresql

	SELECT create_reference_table('nation');

upgrade_to_reference_table
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$
.. _upgrade_to_reference_table:

upgrade_to_reference_table()函数采用分片数目为1的现有分布式表，并将其升级为可识别的引用表。调用此函数后，该表将如同使用:ref:`create_reference_table <create_reference_table>创建一样。

参数
************************

**table_name:** 分布式表的名称（具有分片数目=1），它将作为引用表分布。

返回值
********************************

N/A

示例
*************************

这个示例通知数据库，应该将nation表定义为引用表

.. code-block:: postgresql

	SELECT upgrade_to_reference_table('nation');

.. _mark_tables_colocated:

mark_tables_colocated
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

mark_tables_colocated()函数采用分布式表（源表）和一系列其他表（目标表），并将目标表放入与源表相同的共址组中。如果源表尚未在组中，则此函数会创建一个，并将源表和目标表分配给它。

通常，应该通过create_distributed_table的``colocate_with``参数在表分布时完成对表的共址处理。但必要时，mark_tables_colocated可以解决它。

参数
************************

**source_table_name:** 分布表的名称，目标表将分配给与之匹配的共址组。

**target_table_names:** 分布式目标表的名称数组，必须为非空。这些分布式表必须与以下源表相匹配：

  * 分布方法
  * 分布列类型
  * 复制类型
  * 分片数目

如果做不到这一点，Citus将引发错误。例如，尝试共址``apples``和``oranges``表, 它们的分布列列类型不同会导致：

::

  ERROR:  XX000: cannot colocate tables apples and oranges
  DETAIL:  Distribution column types don't match for apples and oranges.

返回值
********************************

N/A

示例
*************************

本实施例将``products``和``line_items``放入与``stores``相同的共址组。该示例假定这些表都分布在具有匹配类型的列上，很可能是"store id."。

.. code-block:: postgresql

  SELECT mark_tables_colocated('stores', ARRAY['products', 'line_items']);

master_create_distributed_table
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$
.. _master_create_distributed_table:

.. 注意::
   这个函数是已经废弃，并被:ref:`create_distributed_table <create_distributed_table>`代替。

master_create_distributed_table()函数用于定义分布式表。此函数接受表名，分步列和分步方法，并插入适当的元数据用于将表标记为分布式。

参数
************************

**table_name:** 需要分布的表的名称。

**distribution_column:** 要分布该表的列。

**distribution_method:** 要分步该表的方法。允许的值是append或hash。

返回值
********************************

N/A

示例
*************************

此示例通知数据库github_events表应该在repo_id列上使用hash分布。

.. code-block:: postgresql

	SELECT master_create_distributed_table('github_events', 'repo_id', 'hash');


master_create_worker_shards
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$
.. _master_create_worker_shards:

.. 注意::

   这个函数已经废弃，已经被:ref:`create_distributed_table <create_distributed_table>`代替。

master_create_worker_shards()函数使用所需复制因子为*hash*分布式的表创建指定数量的工作分片。当这样做时，该函数还为每个分片分配一部分散列令牌空间(跨越-2亿到20亿之间)。一旦创建了所有碎片，此功能会将所有分布式元数据保存在协调者上。

参数
*****************************

**table_name:** 要为其创建分片的哈希分布表的名称。

**shard_count:** 要创建的分片数。

**replication_factor:** 每个分片所需的复制因子。

返回值
**************************
N/A

示例
***************************

此示例用法将为github_events表创建总共16个分片，其中每个分片拥有散列令牌空间的一部分并在2个worker上复制。

.. code-block:: postgresql

	SELECT master_create_worker_shards('github_events', 16, 2);


master_create_empty_shard
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

master_create_empty_shard()函数可用于为*append* 分布式表创建空分片。在幕后，函数首先选择 shard_replication_factor 工作者来创建分片。然后，它连接到工作者并在选定的工作者上创建分片的空位置。最后，在协调者上更新这些展示位置的元数据，使这些分片在将来的查询中可见。如果无法创建所需数量的分片展示位置，则该函数会出错。

参数
*********************

**table_name:** 要为其创建新分片的append分布式表的名称。

返回值
****************************

**shard_id:** 该函数返回分配给新创建的分片的唯一ID。

示例
**************************

此示例为github_events表创建一个空分片。创建的分片的ID是102089。

.. code-block:: postgresql

    SELECT * from master_create_empty_shard('github_events');
     master_create_empty_shard
    ---------------------------
                    102089
    (1 row)

表和分片 DML
-------------------

.. _master_append_table_to_shard:

master_append_table_to_shard
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

master_append_table_to_shard()函数可用于将PostgreSQL表的内容附加到*append*分布式表的分片。在幕后，该函数连接到具有该分片落点的每个工作者，并将表的内容附加到每个分片。然后，该函数根据每个添加成功或失败的方式更新分片落点的元数据。

如果该函数能够成功附加到至少一个分片落点，则该函数将成功返回。它还会将附加失败的任何落点标记为INACTIVE，以便将来的任何查询都不会考虑该落点。如果所有落点的的附加都失败，则该函数将退出并显示错误（因为未附加任何数据）。在这种情况下，元数据保持不变。

参数
************************

**shard_id:** 切分的Id, 表的内容将被附加到它。

**source_table_name:** PostgreSQL表的名称, 它的内容将被附加。

**source_node_name:** 源表所在节点的DNS名称(“源”节点)。

**source_node_port:** 数据库服务器正在监听的源工作节点上的端口。

返回值
****************************

**shard_fill_ratio:** 该函数返回分片的填充率，它定义为当前分片大小与配置参数shard_max_size的比率。

示例
******************

本例将github_events_local表的内容附加到id为102089的分片中。表github_events_local出现在端口号为5432的节点master-101上运行的数据库中。该函数返回当前分片大小与最大分片大小的比例，0.1表示已填充10％的分片。

.. code-block:: postgresql

    SELECT * from master_append_table_to_shard(102089,'github_events_local','master-101', 5432);
     master_append_table_to_shard
    ------------------------------
                     0.100548
    (1 row)


master_apply_delete_command
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

master_apply_delete_command()函数用于删除与*append*分布式表上的delete命令指定的条件匹配的分片。仅当分片中的所有行都与删除条件匹配时，此函数才会删除分片。由于该函数使用分片元数据来决定是否需要删除分片，因此它要求DELETE语句中的WHERE子句位于分布列上。如果未指定条件，则删除该表的所有分片。

在幕后，此函数连接到具有与删除条件匹配的分片的所有工作节点，并向它们发送一条命令删除所选分片。然后，该函数更新协调者上的相应元数据。如果该函数能够成功删除分片落点，则会删除其元数据。如果无法删除特定落点，则会将其标记为“删除”。标记为“删除”的落点不会考虑用于将来的查询，可以在以后进行清理。

参数
*********************

**delete_command:** 有效的 `SQL DELETE <http://www.postgresql.org/docs/current/static/sql-delete.html>`_ 命令

返回值
**************************

**deleted_shard_count:** 该函数返回与条件匹配并被删除（或标记为删除）的分片数。请注意，这是分片的数量，而不是分片落点的数量。

示例
*********************

第一个示例删除github_events表的所有分片，因为未指定删除条件。在第二个示例中，仅删除与条件匹配的分片（在这种情况下为3）。

.. code-block:: postgresql

    SELECT * from master_apply_delete_command('DELETE FROM github_events');
     master_apply_delete_command
    -----------------------------
                               5
    (1 row)
 
    SELECT * from master_apply_delete_command('DELETE FROM github_events WHERE review_date < ''2009-03-01''');
     master_apply_delete_command
    -----------------------------
                               3
    (1 row)

master_modify_multiple_shards
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

master_modify_multiple_shards()函数用于运行可能跨越多个分片的数据修改语句。根据citus.multi_shard_commit_protocol的值，提交可以在一个或两个阶段完成。

限制:

* 它不能在事务块内调用
* 必须仅使用简单的运算符表达式调用它

参数
**********

**modify_query:** 一个简单的DELETE或UPDATE查询字符串。

返回值
************

N/A

示例
********

.. code-block:: postgresql

  SELECT master_modify_multiple_shards(
    'DELETE FROM customer_delete_protocol WHERE c_custkey > 500 AND c_custkey < 500');

元数据/配置信息
------------------------------------------------------------------------

.. _master_add_node:

master_add_node
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

master_add_node()函数在Citus元数据表pg_dist_node中注册集群中添加的新节点。它还将引用表复制到新节点。

参数
************************

**node_name:** 要添加的新节点的DNS名称或IP地址。

**node_port:** PostgreSQL在工作节点上监听的端口。

**group_id:** 一组主服务器和零个或多个辅助服务器，仅与流复制相关。默认值为0

**node_role:** 是'primary'还是'secondary'。默认'primary'

**node_cluster:** 群集名称。默认'default'

返回值
******************************

一个元组，表示来自:ref:`pg_dist_node<pg_dist_node>`表的一行。

示例
***********************

.. code-block:: postgresql

    select * from master_add_node('new-node', 12345);
     nodeid | groupid | nodename | nodeport | noderack | hasmetadata | isactive | groupid | noderole | nodecluster
    --------+---------+----------+----------+----------+-------------+----------+---------+----------+ ------------
          7 |       7 | new-node |    12345 | default  | f           | t        |       0 | primary  | default
    (1 row)

.. _master_update_node:

master_update_node
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

master_update_node()函数更改Citus元数据表:ref:`pg_dist_node <pg_dist_node>`中注册的节点的主机名和端口。

参数
************************

**node_id:** 来自pg_dist_node表的id。

**node_name:** updated DNS name or IP address for the node. 要更新的节点DNS名称或IP地址。

**node_port:** PostgreSQL在工作节点上监听的端口。

返回值
******************************

N/A

示例
***********************

.. code-block:: postgresql

    select * from master_update_node(123, 'new-address', 5432);

.. _master_add_inactive_node:

master_add_inactive_node
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

:code:`master_add_inactive_node`函数类似于:ref:`master_add_node`，在:code:`pg_dist_node`中注册一个新节点。但是，它将新节点标记为非活动状态，这意味着不会在其中放置任何分片。此外，它也*没有*复制引用表到新的节点。

参数
************************

**node_name:** 要添加的新节点的DNS名或IP地址。

**node_port:** PostgreSQL在工作节点上监听的端口。

**group_id:** 一组一个主服务器和零个或多个辅助服务器，仅与流复制相关。默认值为0

**node_role:** 是'primary'或'secondary'. 默认'primary'

**node_cluster:** 群集名称。默认'default'

返回值
******************************

一个元组，表示来自:ref:`pg_dist_node <pg_dist_node>`表的一行。

示例
***********************

.. code-block:: postgresql

    select * from master_add_inactive_node('new-node', 12345);
     nodeid | groupid | nodename | nodeport | noderack | hasmetadata | isactive | groupid | noderole | nodecluster
    --------+---------+----------+----------+----------+-------------+----------+---------+----------+ -------------
          7 |       7 | new-node |    12345 | default  | f           | f        |       0 | primary  | default
    (1 row)

master_activate_node
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

:code:`master_activate_node`函数将节点在Citus元数据表:code:`pg_dist_node`中标记为活动节点，并将引用表复制到节点。对通过master_add_inactive_node添加的节点很有用。

参数
************************

**node_name:** 要添加的新节点的DNS名称或IP地址。

**node_port:**  PostgreSQL在工作节点上侦听的端口。

返回值
******************************

一个元组，表示来自:ref:`pg_dist_node<pg_dist_node>`表的一行。

示例
***********************

.. code-block:: postgresql

    select * from master_activate_node('new-node', 12345);
     nodeid | groupid | nodename | nodeport | noderack | hasmetadata | isactive| noderole | nodecluster
    --------+---------+----------+----------+----------+-------------+---------+----------+ -------------
          7 |       7 | new-node |    12345 | default  | f           | t       | primary  | default
    (1 row)

master_disable_node
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

:code:`master_disable_node`函数是相反的 master_activate_node。它在Citus元数据表:code:`pg_dist_node`中将节点标记为非活动状态，暂时将其从群集中删除。该功能还会从已禁用的节点中删除所有参考表落点。要重新激活节点，只要再次运行:code:`master_activate_node`。

参数
************************

**node_name:** 要禁用的节点的DNS名称或IP地址。

**node_port:** PostgreSQL在工作节点上侦听的端口。

返回值
******************************

N/A

示例
***********************

.. code-block:: postgresql

    select * from master_disable_node('new-node', 12345);

.. _master_add_secondary_node:

master_add_secondary_node
$$$$$$$$$$$$$$$$$$$$$$$$$

master_add_secondary_node()函数在集群中为现有主节点注册新的辅助节点。它更新了Citus元数据表pg_dist_node。

参数
************************

**node_name:** 要添加的新节点的DNS名称或IP地址。

**node_port:** PostgreSQL在工作节点上侦听的端口。

**primary_name:** 此辅助节点的主节点的DNS名称或IP地址。

**primary_port:** PostgreSQL在主节点上侦听的端口。

**node_cluster:** 群集名称。默认'default'

返回值
******************************

一个元组，表示来自:ref:`pg_dist_node <pg_dist_node>`表的一行。

示例
***********************

.. code-block:: postgresql

    select * from master_add_secondary_node('new-node', 12345, 'primary-node', 12345);
     nodeid | groupid | nodename | nodeport | noderack | hasmetadata | isactive | noderole  | nodecluster
    --------+---------+----------+----------+----------+-------------+----------+-----------+-------------
          7 |       7 | new-node |    12345 | default  | f           | t        | secondary | default
    (1 row)


master_remove_node
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

master_remove_node()函数从pg_dist_node元数据表中删除指定的节点。如果此节点上存在分片落点，则此函数将出错。因此，在使用此功能之前，需要将分片移出该节点。

参数
************************

**node_name:** 要删除的节点的DNS名称。

**node_port:** PostgreSQL在工作节点上侦听的端口。

返回值
******************************

N/A

示例
***********************

.. code-block:: postgresql

    select master_remove_node('new-node', 12345);
     master_remove_node 
    --------------------
     
    (1 row)

master_get_active_worker_nodes
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

master_get_active_worker_nodes()函数返回活动的工作者主机名和端口号的列表。目前，该函数假定pg_dist_node目录表中的所有工作节点都处于活动状态。

参数
************************

N/A

返回值
******************************

每个元组包含以下信息的元组列表：

**node_name:** 工作节点的DNS名称

**node_port:** 数据库服务器正在侦听的工作节点上的端口

示例
***********************

.. code-block:: postgresql

    SELECT * from master_get_active_worker_nodes();
     node_name | node_port 
    -----------+-----------
     localhost |      9700
     localhost |      9702
     localhost |      9701

    (3 rows)

master_get_table_metadata
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

master_get_table_metadata()
函数可用于返回分布式表的与分发相关的元数据。此元数据包括该表的关系ID，存储类型，分布方法，分布列，复本计数，最大分片大小和分片放置策略。在幕后，该函数查询Citus元数据表以获得所需的信息，并在将其返回给用户之前将其连接到一个元组中。

参数
***********************

**table_name:** 要为其获取元数据的分布式表的名称。

返回值
*********************************

包含以下信息的元组：

**logical_relid:** 分布式表的Oid。此值引用pg_class系统目录表中的relfilenode列。

**part_storage_type:** 用于表的存储类型。可能是't'(standard table), 'f'(foreign table)或 'c'(columnar table)。

**part_method:** 表格使用的分布方法。可以是'a'(append)或'h'(hash)。

**part_key:** 表的分布列。

**part_replica_count:** 当前分片复本计数。

**part_max_size:** 当前最大分片大小(以字节为单位)。

**part_placement_policy:** 分片放置策略，用于放置表的分片。可以是1(local-node-first)或2(round-robin)。

示例
*************************

下面的示例获取并显示github_events表的表元数据。

.. code-block:: postgresql

    SELECT * from master_get_table_metadata('github_events');
     logical_relid | part_storage_type | part_method | part_key | part_replica_count | part_max_size | part_placement_policy 
    ---------------+-------------------+-------------+----------+--------------------+---------------+-----------------------
             24180 | t                 | h           | repo_id  |                  2 |    1073741824 |                     2
    (1 row)

.. _get_shard_id:

get_shard_id_for_distribution_column
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

Citus根据行的分布列的值和表的分布方法将分布式表的每一行分配给分片。在大多数情况下，精确的映射是数据库管理员可以忽略的底层细节。但是，确定行的分片可能很有用，既可用于手动数据库维护任务，也可用于满足好奇心。该:code:`get_shard_id_for_distribution_column`函数为hash-和range-分布表以及引用表提供此信息。它不适用于append分布。

参数
************************

**table_name:** 分布式表。

**distribution_value:** 分发列的值。

返回值
******************************

分片ID Citus与给定表的分发列值相关联。
The shard id Citus associates with the distribution column value for the given table.

示例
***********************

.. code-block:: postgresql

  SELECT get_shard_id_for_distribution_column('my_table', 4);

   get_shard_id_for_distribution_column
  --------------------------------------
                                 540007
  (1 row)

column_to_column_name
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

将:code:`pg_dist_partition`的:code:`partkey`列转换为文本列名称。这对于确定分布式表的分发列很有用。

有关更详细的讨论，请参阅:ref:`finding_dist_col`。

参数
************************

**table_name:** 分布式表。

**column_var_text:** :code:`pg_dist_partition` 表中:code:`partkey`列的值.

返回值
******************************

:code:`table_name`'的分发列的名称。

示例
***********************

.. code-block:: postgresql

  -- get distribution column name for products table

  SELECT column_to_column_name(logicalrelid, partkey) AS dist_col_name
    FROM pg_dist_partition
   WHERE logicalrelid='products'::regclass;

Output:

::

  ┌───────────────┐
  │ dist_col_name │
  ├───────────────┤
  │ company_id    │
  └───────────────┘

citus_relation_size
$$$$$$$$$$$$$$$$$$$

获取指定分布式表的所有分片使用的磁盘空间。这包括"main fork,"的大小，但不包括分片的visibility map和free space map。

参数
*********

**logicalrelid:** 分布式表的名称。

返回值
************

以字节为单位的大小。

示例
*******

.. code-block:: postgresql

  SELECT pg_size_pretty(citus_relation_size('github_events'));

::

  pg_size_pretty
  --------------
  23 MB

citus_table_size
$$$$$$$$$$$$$$$$

获取指定分布式表的所有分片使用的磁盘空间，不包括索引（但包括TOAST, free space map, and visibility map）。
Get the disk space used by all the shards of the specified distributed table, excluding indexes (but including TOAST, free space map, and visibility map).

参数
*********

**logicalrelid:** 分布式表的名称。

返回值
************

以字节为单位的大小。

示例
*******

.. code-block:: postgresql

  SELECT pg_size_pretty(citus_table_size('github_events'));

::

  pg_size_pretty
  --------------
  37 MB

citus_total_relation_size
$$$$$$$$$$$$$$$$$$$$$$$$$

获取指定分布式表的所有分片使用的总磁盘空间，包括所有索引和TOAST数据。
Get the total disk space used by the all the shards of the specified distributed table, including all indexes and TOAST data.

参数
*********

**logicalrelid:** 分布式表的名称。

返回值
************

以字节为单位的大小。

示例
*******

.. code-block:: postgresql

  SELECT pg_size_pretty(citus_total_relation_size('github_events'));

::

  pg_size_pretty
  --------------
  73 MB


citus_stat_statements_reset
$$$$$$$$$$$$$$$$$$$$$$$$$$$

从:ref:`citus_stat_statements <citus_stat_statements>`中删除所有行。请注意，这独立于``pg_stat_statements_reset()``。要重置所有统计数据，请调用这两个函数。

参数
*********

N/A

返回值
************

None

.. _cluster_management_functions:

集群管理和修复功能
----------------------------------------

master_copy_shard_placement
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

如果在修改命令或DDL操作期间无法更新分片位置，则会将其标记为非活动状态。然后可以使用来自健康位置的数据调用master_copy_shard_placement函数来修复非活动的分片位置。

要修复分片，该函数首先删除不健康的分片位置并使用协调器上的模式重新创建它。创建分片位置后，该函数将从正常位置中复制数据并更新元数据，以将新分片放置标记为正常。此功能可确保在修复期间保护分片不受任何并发修改的影响。

参数
**********

**shard_id:** 要修复的分片的ID。

**source_node_name:** 存在健康分片位置的节点的DNS名称("source" 节点)。

**source_node_port:** 数据库服务器正在侦听的源工作节点上的端口。

**target_node_name:** 存在无效分片位置的节点的DNS名称("target"节点)。

**target_node_port:** 数据库服务器正在侦听的目标工作节点上的端口。

返回值
************

N/A

示例
********

下面的示例将修复shard 12345的非活动分片位置，该分片位于'bad_host'数据库服务器上, 端口5432。要修复它，它将使用'good_host'服务器上存在的健康分片放置中的数据, 端口5432。

.. code-block:: postgresql

    SELECT master_copy_shard_placement(12345, 'good_host', 5432, 'bad_host', 5432);

master_move_shard_placement
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

.. 注意::

  master_move_shard_placement函数是Citus Enterprise的一部分。请`联系我们 <https://www.citusdata.com/about/contact_us>`_ 获取此功能。

此函数将给定的分片（以及与之共址的分片）从一个节点移动到另一个节点。它通常在分片重新平衡期间间接使用，而不是由数据库管理员直接调用。

有两种方法可以移动数据：阻塞或非阻塞。阻塞方法意味着在移动期间暂停对分片的所有修改。第二种方法，它避免阻止分片写入，依赖于Postgres 10的逻辑复制。

成功移动操作后，源节点中的分片将被删除。如果移动在任何时间点失败，则此函数会抛出错误并使源节点和目标节点保持不变。

参数
**********

**shard_id:** 要移动的分片的ID。

**source_node_name:** 存在健康分片位置的节点的DNS名称（“源”节点）。

**source_node_port:** 数据库服务器正在侦听的源工作节点上的端口。

**target_node_name:** 存在无效分片位置的节点的DNS名称（“目标”节点）。

**target_node_port:** 数据库服务器正在侦听的目标工作节点上的端口。

**shard_transfer_mode:** (可选) 指定复制方法，是否使用PostgreSQL逻辑复制或跨工作者COPY命令。可能的值是：
Specify the method of replication, whether to use PostgreSQL logical replication or a cross-worker COPY command. The possible values are:

  * ``auto``: 如果可以进行逻辑复制，则需要副本标识，否则使用旧行为（例如，用于分片修复，PostgreSQL 9.6）。这是默认值。
  * ``force_logical``: 即使表没有副本标识，也请使用逻辑复制。在复制期间，对表的任何并发更新/删除语句都将失败。
  * ``block_writes``: 对缺少主键或副本标识的表使用COPY（阻止写入）。

返回值
************

N/A

示例
********

.. code-block:: postgresql

    SELECT master_move_shard_placement(12345, 'from_host', 5432, 'to_host', 5432);

.. _rebalance_table_shards:

rebalance_table_shards
$$$$$$$$$$$$$$$$$$$$$$$$$$$

.. 注意::
  rebalance_table_shards 函数是Citus Enterprise的一部分。请`联系我们 <https://www.citusdata.com/about/contact_us>`_ 获取此功能。

rebalance_table_shards()函数移动给定表的分片，使它们在工作者之间均匀分布。
该函数首先计算它需要进行的移动列表，以确保集群在给定阈值内保持平衡。
然后，它将分片位置从源节点逐个移动到目标节点，并更新相应的分片元数据以反映移动。

参数
**************************

**table_name:** 需要重新平衡其分片的表的名称。

**threshold:** (可选)介于0.0和1.0之间的浮点数，表示节点利用率与平均利用率的最大差异比率。例如，指定0.1将导致分片重新平衡器尝试平衡所有节点以保持相同数量的分片±10％。具体来说，分片重新平衡器将尝试将所有工作节点的利用率收敛到(1 - threshold) * average_utilization ... (1 + threshold) * average_utilization 范围。

**max_shard_moves:** (可选)要移动的最大分片数。

**excluded_shard_list:** (可选)在重新平衡操作期间不应移动的分片的标识符。

**shard_transfer_mode:** (可选)指定复制方法，是否使用PostgreSQL逻辑复制或跨工作者COPY命令。可能的值是：

  * ``auto``: 如果可以进行逻辑复制，则需要副本标识，否则使用旧行为（例如，用于分片修复，PostgreSQL 9.6）。这是默认值。
  * ``force_logical``: 即使表没有副本标识，也请使用逻辑复制。在复制期间，对表的任何并发更新/删除语句都将失败。
  * ``block_writes``: 对缺少主键或副本标识的表使用COPY（阻止写入）。

返回值
*********************************

N/A

示例
**************************

以下示例将尝试在默认阈值内重新平衡github_events表的分片。

.. code-block:: postgresql

	SELECT rebalance_table_shards('github_events');

此示例用法将尝试重新平衡github_events表，而不移动ID为1和2的分片。

.. code-block:: postgresql

	SELECT rebalance_table_shards('github_events', excluded_shard_list:='{1,2}');

.. _get_rebalance_progress:

get_rebalance_progress
$$$$$$$$$$$$$$$$$$$$$$

.. 注意::

  get_rebalance_progress()函数是Citus Enterprise的一部分。请`联系我们 <https://www.citusdata.com/about/contact_us>`_ 获取此功能。

一旦分片重新平衡开始，该``get_rebalance_progress()``函数将列出所涉及的每个分片的进度。它监视移动计划和用``rebalance_table_shards()``执行。

参数
**************************

N/A

返回值
*********************************

元组, 包含这些列：

* **sessionid**: 重新平衡监视器的Postgres PID
* **table_name**: 分片正在移动的表
* **shardid**: 有问题的分片
* **shard_size**: 大小（以字节为单位）
* **sourcename**: 源节点的主机名
* **sourceport**: 源节点的端口
* **targetname**: 目标节点的主机名
* **targetport**: 目标节点的端口
* **progress**: 0 = 等待移动; 1 = 移动中; 2 = 完成

示例
**************************

.. code-block:: sql

  SELECT * FROM get_rebalance_progress();

::

  ┌───────────┬────────────┬─────────┬────────────┬───────────────┬────────────┬───────────────┬────────────┬──────────┐
  │ sessionid │ table_name │ shardid │ shard_size │  sourcename   │ sourceport │  targetname   │ targetport │ progress │
  ├───────────┼────────────┼─────────┼────────────┼───────────────┼────────────┼───────────────┼────────────┼──────────┤
  │      7083 │ foo        │  102008 │    1204224 │ n1.foobar.com │       5432 │ n4.foobar.com │       5432 │        0 │
  │      7083 │ foo        │  102009 │    1802240 │ n1.foobar.com │       5432 │ n4.foobar.com │       5432 │        0 │
  │      7083 │ foo        │  102018 │     614400 │ n2.foobar.com │       5432 │ n4.foobar.com │       5432 │        1 │
  │      7083 │ foo        │  102019 │       8192 │ n3.foobar.com │       5432 │ n4.foobar.com │       5432 │        2 │
  └───────────┴────────────┴─────────┴────────────┴───────────────┴────────────┴───────────────┴────────────┴──────────┘

replicate_table_shards
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

.. 注意::
  replicate_table_shards 函数是Citus Enterprise的一部分。请`联系我们 <https://www.citusdata.com/about/contact_us>`_ 获取此功能。

replicate_table_shards()函数复制给定表的未复制的分片。该函数首先计算未复制的分片列表以及从中获取它们以进行复制的位置。然后，该函数复制这些分片并更新相应的分片元数据以反映副本。

参数
*************************

**table_name:** 需要复制其分片的表的名称。

**shard_replication_factor:** (可选)为每个分片获得的所需复制因子。

**max_shard_copies:** (可选)要复制以达到所需复制因子的最大分片数。

**excluded_shard_list:** (可选)在复制操作期间不应复制的分片标识符。

返回值
***************************

N/A

示例s
**************************

下面的示例将尝试将github_events表的分片复制到shard_replication_factor。

.. code-block:: postgresql

	SELECT replicate_table_shards('github_events');

此示例将尝试将github_events表的分片带到所需的复制因子，最多包含10个分片副本。这意味着重新平衡器在尝试达到所需的复制因子时，最多只能复制10个分片。

.. code-block:: postgresql

	SELECT replicate_table_shards('github_events', max_shard_copies:=10);

.. _isolate_tenant_to_new_shard:

isolate_tenant_to_new_shard
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

.. 注意::
  isolate_tenant_to_new_shard函数是Citus Enterprise的一部分。请`联系我们 <https://www.citusdata.com/about/contact_us>`_ 获取此功能。

这个函数创建一个新的切分来保存分布列中具有特定单个值的行。对于多租户Citus用例来说尤其方便，其中大租户可以单独放置在自己的分片上，最终放置在自己的物理节点上。

有关更深入的讨论，请参阅:ref:`tenant_isolation`。

参数
*************************

**table_name:** 获取新分片的表的名称。

**tenant_id:** 将分配给新分片的分布列的值。

**cascade_option:** (可选)当设置为"CASCADE,"时，还会将分片与当前表的:ref:`colocation_groups`中的所有表隔离。

返回值
***************************

**shard_id:** 该函数返回分配给新创建的分片的唯一ID。

示例s
**************************

创建一个新的分片以保存租户135的lineitems：

.. code-block:: postgresql

  SELECT isolate_tenant_to_new_shard('lineitem', 135);

::

  ┌─────────────────────────────┐
  │ isolate_tenant_to_new_shard │
  ├─────────────────────────────┤
  │                      102240 │
  └─────────────────────────────┘

citus_create_restore_point
$$$$$$$$$$$$$$$$$$$$$$$$$$

暂时阻止写入群集，并在所有节点上创建命名还原点。此函数类似于`pg_create_restore_point <https://www.postgresql.org/docs/10/static/functions-admin.html#FUNCTIONS-ADMIN-BACKUP>`_，但适用于所有节点，并确保还原点在它们之间保持一致。此功能非常适合进行时间点恢复和群集分叉。

参数
*************************

**name:** 要创建的还原点的名称。

返回值
***************************

**coordinator_lsn:** 协调器节点WAL中的还原点的日志序列号。

示例
**************************

.. code-block:: postgresql

  select citus_create_restore_point('foo');

::

  ┌────────────────────────────┐
  │ citus_create_restore_point │
  ├────────────────────────────┤
  │ 0/1EA2808                  │
  └────────────────────────────┘
