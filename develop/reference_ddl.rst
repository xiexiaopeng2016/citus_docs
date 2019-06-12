.. _ddl:

创建和修改分布式表（DDL）
==============================

.. note::

   Citus (包括 :ref:`mx`) 要求仅从协调者节点运行DDL命令

创建和分布表
-----------------

要创建分布式表，首先需要定义表模式。为此，您可以使用 `CREATE TABLE <http://www.postgresql.org/docs/current/static/sql-createtable.html>`_ 语句以与使用常规PostgreSQL表相同的方式定义表。

.. code-block:: sql

    CREATE TABLE github_events
    (
    	event_id bigint,
    	event_type text,
    	event_public boolean,
    	repo_id bigint,
    	payload jsonb,
    	repo jsonb,
    	actor jsonb,
    	org jsonb,
    	created_at timestamp
    );

接下来，您可以使用create_distributed_table（）函数指定表分布列并创建工作分片。

.. code-block:: sql

    SELECT create_distributed_table('github_events', 'repo_id');

此函数通知Citus应该在repo_id列上分发github_events表（通过散列列值）。该函数还使用citus.shard_count和citus.shard_replication_factor配置值在工作者节点上创建分片。

此示例将创建总共citus.shard_count个分片数，其中每个分片拥有散列令牌空间的一部分，并根据默认的citus.shard_replication_factor配置值进行复制。在worker上创建的分片副本具有与协调者上的表相同的表模式，索引和约束定义。创建副本后，此函数会将所有分布式元数据保存在协调者上。

每个创建的分片都分配了一个唯一的分片ID，并且其所有副本都具有相同的分片ID。每个分片在工作者节点上表示为名为“tablename_shardid”的常规PostgreSQL表，其中tablename是分布式表的名称，shardid是分配给该分片的唯一ID。您可以连接到工作者节点postgres实例以查看或运行各个分片上的命令。

您现在可以将数据插入到分布式表中并对其运行查询。您还可以在我们文档的Citus实用程序功能中了解有关本节中使用的
:ref:`user_defined_functions` 的更多信息。

.. _reference_tables:

引用表
~~~~~~~~~~

上述方法将表分配到多个水平分片中，但另一种可能性是将表分配到单个分片中并将分片复制到每个工作者节点。以这种方式分发的表称为引用表。它们用于存储需要由群集中的多个节点频繁访问的数据。

常用的引用表包括:

* 需要与较大的分布式表连接的较小表。
* 多租户应用中缺少租户ID列或与租户无关的表。（在某些情况下，为了减少迁移工作量，用户甚至可能选择从与租户关联的表中创建引用表，但这些表当前缺少租户ID。）
* 表需要跨多个列的唯一约束并且足够小。

例如，假设多租户电子商务网站需要计算其任何商店中的交易的销售税。税务信息并非特定于任何租户。将它合并到共享表中是有意义的。以美国为中心的引用表可能如下所示：

.. code-block:: postgresql

  -- a reference table

  CREATE TABLE states (
    code char(2) PRIMARY KEY,
    full_name text NOT NULL,
    general_sales_tax numeric(4,3)
  );

  -- 将其分布给所有工作者节点

  SELECT create_reference_table('states');

现在，诸如购物车的一个计算税的查询可以在没有网络开销的情况下连接 :code:`states` 表，并且可以向州代码添加外键以便更好地进行验证。

除了将表分布为单个复制的分片之外，:code:`create_reference_table` UDF 还将其标记为Citus元数据表中的引用表。Citus自动执行两阶段提交(`2PC <https://en.wikipedia.org/wiki/Two-phase_commit_protocol>`_)以修改以这种方式标记的表，从而提供强大的一致性保证。

如果现有分布式表的分片计数为1，则可以通过运行将其升级为可识别的引用表

.. code-block:: postgresql

  SELECT upgrade_to_reference_table('table_name');

有关在多租户应用程序中使用引用表的另一个示例，请参阅 :ref:`mt_ref_tables`。

分布协调者数据
~~~~~~~~~~~~~~~~~~~

如果将现有的PostgreSQL数据库转换为Citus集群的协调者节点，则可以有效地分发其表中的数据，并且对应用程序的中断最小。

:code:`create_distributed_table` 前面描述的函数适用于空表和非空表，对于后者，它会自动在整个集群中分配表行。您将知道它是否通过消息的存在来执行此操作，“注意：从本地表复制数据...”例如：
The :code:`create_distributed_table` function described earlier works on both empty and non-empty tables, and for the latter it automatically distributes table rows throughout the cluster. You will know if it does this by the presence of the message, "NOTICE:  Copying data from local table..." For example:

.. code-block:: postgresql

  CREATE TABLE series AS SELECT i FROM generate_series(1,1000000) i;
  SELECT create_distributed_table('series', 'i');
  NOTICE:  Copying data from local table...
   create_distributed_table
   --------------------------

   (1 row)

迁移数据时会阻止表上的写入，并且一旦函数提交，挂起的写入将作为分布式查询处理。（如果函数失败，则查询将再次变为本地。）读取可以正常继续，并在函数提交后成为分布式查询。

.. note::

  当在它们之间分配许多具有外键的表时，最好在运行 :code:`create_distributed_table` 之前删除外键, 并在分发表之后重新创建它们。当一个表是分布式的，而另一个表不是分布式的时，不能总是强制执行外键。但是外键在分布表和参考表之间*都*支持。

当将数据从外部数据库(如Amazon RDS)迁移到Citus Cloud时，首先通过:code: ' create_distributed_table '创建Citus分布式表，然后将数据复制到表中

.. _colocation_groups:

共同定位表
---------------

协同定位是在战术上划分数据，在相同机器上保留相关信息以实现有效的关系操作，同时利用整个数据集的水平可伸缩性的实践。有关更多信息和示例，请参阅 :ref:`colocation`。

表共同定位于组中。要手动控制表的共址组分配，请使用 :code:`create_distributed_table` 的可选参数 :code:`colocate_with` 。如果您不关心表的共址，则省略此参数。它默认为该值'default'，该值将表与具有相同分发列类型，分片计数和复制因子的任何其他默认协同定位表分组。

.. code-block:: postgresql

  -- 这些表通过使用相同的
  -- 分布列类型和具有默认
  -- 共址组的分片计数隐式地共同定位

  SELECT create_distributed_table('A', 'some_int_col');
  SELECT create_distributed_table('B', 'other_int_col');

如果新表与其可能的隐式共址组中的其他表无关，请指定 :code:`colocated_with => 'none'` 。

.. code-block:: postgresql

  -- 与其他表不在同一共址

  SELECT create_distributed_table('A', 'foo', colocate_with => 'none');

将不相关的表拆分到它们自己的共址组中将改善分片 :ref:`重新平衡 <shard_rebalancing>` 性能，因为同一组中的分片必须一起移动。

当表确实相关时（例如，当它们将被连接时），明确地共同定位它们是有意义的。适当的共址的收益比任何重新平衡开销都重要。

要明确地共同定位多个表，请分配一个表，然后将其他表放入其共址组。例如：

.. code-block:: postgresql

  -- 分布 stores
  SELECT create_distributed_table('stores', 'store_id');

  -- 添加到与stores相同的组中
  SELECT create_distributed_table('orders', 'store_id', colocate_with => 'stores');
  SELECT create_distributed_table('products', 'store_id', colocate_with => 'stores');

有关共址组的信息存储在 :ref:`pg_dist_colocation <colocation_group_table>` 表中，而 :ref:`pg_dist_partition <partition_table>` 显示哪些表分配给哪些组。

.. _marking_colocation:

从Citus 5.x升级
~~~~~~~~~~~~~~~

从Citus 6.0开始，我们将co-location设置为一等的概念，并开始在pg_dist_colocation中跟踪表对共址组的分配。由于Citus 5.x没有这个概念，因此使用Citus 5创建的表没有明确标记为共存于元数据中，即使这些表格在物理上位于同一位置。

由于Citus使用协同定位元数据信息进行查询优化和下推，因此向Citus通知此先前创建的表的共址非常重要。要修复元数据，只需使用mark_tables_colocated将表标记为co-located：

.. code-block:: postgresql

  -- 假设 stores, products and line_items 是在Citus 5.x数据库中创建的.

  -- 将products和line_items放入stores的共址组
  SELECT mark_tables_colocated('stores', ARRAY['products', 'line_items']);

此函数要求使用相同的方法，列类型，分片数和复制方法分步表。它不会重新分片或物理移动数据，它只是更新Citus元数据。

删除表
---------

您可以使用标准PostgreSQL DROP TABLE命令删除分布式表。与常规表一样，DROP TABLE删除目标表存在的所有索引，规则，触发器和约束。此外，它还会删除工作者节点上的分片并清除其元数据。

.. code-block:: sql

    DROP TABLE github_events;

.. _ddl_prop_support:

修改表
---------

Citus自动传播多种DDL语句，这意味着在协调者节点上修改分布式表也会更新工作者的分片。其他DDL语句需要手动传播，而某些其他DDL语句则是禁止的，例如那些会修改分发列的语句。尝试运行不符合自动传播条件的DDL将引发错误并使协调者节点上的表保持不变。

以下是传播的DDL语句类别的参考。请注意，可以使用 :ref:`配置参数 <enable_ddl_prop>` 启用或禁用自动传播。

添加/修改列
~~~~~~~~~~~~~~~~

Citus 自动传播大多数 `ALTER TABLE <https://www.postgresql.org/docs/current/static/ddl-alter.html>`_ 命令。添加列或更改其默认值的工作方式与在单机PostgreSQL数据库中的工作方式相同：

.. code-block:: postgresql

  -- 添加一列

  ALTER TABLE products ADD COLUMN description text;

  -- 更改默认值

  ALTER TABLE products ALTER COLUMN price SET DEFAULT 7.77;

对现有列进行重大更改(如重命名列或更改其数据类型)也可以。但是，不能更改分布列的数据类型。此列确定表数据如何通过Citus群集分发，并且修改其数据类型需要移动数据。

试图这样做会导致错误：

.. code-block:: postgres

  -- assumining store_id is the distribution column
  -- for products, and that it has type integer

  ALTER TABLE products
  ALTER COLUMN store_id TYPE text;

  /*
  ERROR:  XX000: 无法执行涉及分区列的ALTER TABLE命令
  LOCATION:  ErrorIfUnsupportedAlterTableStmt, multi_utility.c:2150
  */

添加/删除约束
~~~~~~~~~~~~~~~~~~

使用Citus可以让您继续享受关系数据库的安全性，包括数据库约束（请参阅PostgreSQL `文档 <https://www.postgresql.org/docs/current/static/ddl-constraints.html>`_）。由于分布式系统的性质，Citus不会交叉引用工作者节点之间的唯一性约束或参照完整性。

在这些情况下可能会创建外键：

* 在两个本地（非分布式）表之间，
* 当键包含分发列时，在两个 :ref:`共置 <colocation>` 的分布式表之间，或
* 作为一个分布式表引用一个 :ref:`引用表 <reference_tables>`

不支持将引用表作为外键约束的*引用*表，即不支持从引用到引用和从引用到分布式的键。

要在共置的分布式表之间设置外键，请始终在键中包含分发列。这可能涉及制作键组合。

.. note::

  主键和唯一性约束必须包括分发列。将它们添加到非分发列将生成错误（请参阅 :ref:`non_distribution_uniqueness`）。

此示例显示如何在分布式表上创建主键和外键：

.. code-block:: postgresql

  --
  -- 添加主键
  -- --------------------

  --我们将在account_id上分发这些表。ads和clicks
  -- 表必须使用包含account_id的复合键。

  ALTER TABLE accounts ADD PRIMARY KEY (id);
  ALTER TABLE ads ADD PRIMARY KEY (account_id, id);
  ALTER TABLE clicks ADD PRIMARY KEY (account_id, id);

  -- 接下来分布表

  SELECT create_distributed_table('accounts', 'id');
  SELECT create_distributed_table('ads',      'account_id');
  SELECT create_distributed_table('clicks',   'account_id');

  --
  -- 添加外键
  -- -------------------

  -- 请注意，这可以在分发之前或之后发生，只要
  -- 目标列上存在唯一性约束
  -- 只能在分发之前强制执行。

  ALTER TABLE ads ADD CONSTRAINT ads_account_fk
    FOREIGN KEY (account_id) REFERENCES accounts (id);
  ALTER TABLE clicks ADD CONSTRAINT clicks_ad_fk
    FOREIGN KEY (account_id, ad_id) REFERENCES ads (account_id, id);

同样，在唯一性约束中包含分发列：

.. code-block:: postgresql

  -- 假设我们希望每个广告都使用独特的图片。请注意，我们可以
  -- 当我们按帐户id分布时，仅对每个帐户强制执行它。

  ALTER TABLE ads ADD CONSTRAINT ads_unique_image
    UNIQUE (account_id, image_url);

非空约束可以应用于任何列（分发或不分发），因为它们不需要在worker之间进行查找。

.. code-block:: postgresql

  ALTER TABLE ads ALTER COLUMN image_url SET NOT NULL;

使用NOT VALID约束
~~~~~~~~~~~~~~~~~

在某些情况下，对新行强制执行约束可能很有用，同时允许现有的不符合行保持不变。Citus使用PostgreSQL的“NOT VALID”约束指定支持CHECK约束和外键的此功能。

例如，考虑将用户配置文件存储在 :ref:`引用表 <reference_tables>` 中的应用程序。

.. code-block:: postgres

   -- 我们在这里使用“text”列类型，但是一个真正的应用程序
   -- 可能使用postgres contrib模块中提供的 `citext<https://www.postgresql.org/docs/current/citext.html>`_

   CREATE TABLE users ( email text PRIMARY KEY );
   SELECT create_reference_table('users');

随着时间的推移想象一些不正确的地址进入表中。

.. code-block:: postgres

   INSERT INTO users VALUES
      ('foo@example.com'), ('hacker12@aol.com'), ('lol');

我们想验证地址，但PostgreSQL通常不允许我们添加对现有行失败的CHECK约束。但是它*确实*允许标记为无效的约束：

.. code-block:: postgres

   ALTER TABLE users
   ADD CONSTRAINT syntactic_email
   CHECK (email ~
      '^[a-zA-Z0-9.!#$%&''*+/=?^_`{|}~-]+@[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$'
   ) NOT VALID;

这成功了，新行受到保护。

.. code-block:: postgres

   INSERT INTO users VALUES ('fake');

   /*
   ERROR:  new row for relation "users_102010" violates
           check constraint "syntactic_email_102010"
   DETAIL:  Failing row contains (fake).
   */

稍后，在非高峰时段，数据库管理员可以尝试修复错误的行并重新验证约束。

.. code-block:: postgres

   -- 稍后，尝试验证所有行
   ALTER TABLE users
   VALIDATE CONSTRAINT syntactic_email;

PostgreSQL文档在 `ALTER TABLE <https://www.postgresql.org/docs/current/sql-altertable.html>`_ 部分中提供了有关NOT VALID和VALIDATE CONSTRAINT的更多信息。

添加/删除索引
~~~~~~~~~~~~~~~~~~~

Citus支持添加和删除 `索引 <https://www.postgresql.org/docs/current/static/sql-createindex.html>`_：

.. code-block:: postgresql

  -- 添加索引

  CREATE INDEX clicked_at_idx ON clicks USING BRIN (clicked_at);

  -- 删除索引

  DROP INDEX clicked_at_idx;

添加索引需要写入锁定，这在多租户“记录系统”中可能是不合需要的。为了最小化应用程序停机时间，请创建 `concurrently <https://www.postgresql.org/docs/current/static/sql-createindex.html#SQL-CREATEINDEX-CONCURRENTLY>`_ 索引代替。此方法比标准索引构建需要更多的总工作，并且需要更长的时间才能完成。但是，由于它允许在构建索引时继续正常操作，因此此方法对于在生产环境中添加新索引很有用。

.. code-block:: postgresql

  -- 添加索引而不锁定表写入

  CREATE INDEX CONCURRENTLY clicked_at_idx ON clicks USING BRIN (clicked_at);

手动修改
~~~~~~~~~~~~

目前，其他DDL命令不会自动传播，但您可以手动传播更改。请参见:ref:`manual_prop`。
