.. _distributed_data_modeling:

选择分布列
===============


Citus使用分布式表中的分布列将表行分配给分片。为每个表选择分布列是 **最重要的** 建模决策之一，因为它决定了数据在节点之间的分布方式。

如果正确选择了分布列，则相关数据将在同一物理节点上组合在一起，从而快速进行查询并添加对所有SQL功能的支持。如果选择的列不正确，系统将不必要地运行缓慢，并且无法跨节点支持所有SQL功能。

本节提供了两种最常见的Citus方案的分布列提示。最后，我们深入研究了“co-location”，即节点上理想的数据分组。

.. _distributing_by_tenant_id:

多租户应用
--------------

多租户架构使用一种分层数据库建模形式，在分布式集群中的节点之间分布查询。数据层次结构的顶部称为租户ID，需要存储在每个表的列中。Citus检查查询以查看它们涉及哪个租户ID，并将查询路由到单个工作者节点进行处理，特别是保存与租户ID关联的数据分片的节点。运行放置在同一节点上的所有相关数据的查询称为 :ref:`colocation`。

下图说明了多租户数据模型中的共址。它包含两个表，Accounts和Campaigns，每个表用 :code:`account_id` 分布。阴影框表示分片，每个分片的颜色代表哪个工作者节点包含它。绿色分片一起存储在一个工作者节点上，蓝色存储在另一个工作者节点上。请注意，在将两个表限制为同一个account_id时，Accounts和Campaigns之间的联接查询如何在一个节点上将所有必要的数据放在一起。

.. figure:: ../images/mt-colocation.png
   :alt: co-located tables in multi-tenant architecture


要在您自己的架构中应用此设计，第一步是确定应用程序中租户的构成。常见实例包括公司，帐户，组织或客户。列名称将类似于 :code:`company_id` 或 :code:`customer_id`。检查每个查询并问自己：如果它有额外的WHERE子句来限制涉及具有相同租户ID的行的所有表，它会工作吗？多租户模型中的查询通常作用于租户，例如，销售或库存上的查询将限定在某个商店中。

最佳实践
^^^^^^^^^^^^

* **通过公共tenant_id列对分布式表进行分区。**
  例如，在租户是公司的SaaS应用程序中，tenant_id可能是company_id。

* **将小型跨租户表转换为引用表。**
  当多个租户共享一个小型信息表时，将其作为 :ref:`引用表 <reference_tables>` 进行分布。

* **通过tenant_id限制过滤所有应用程序查询。**
  每个查询应一次请求一个租户的信息。

阅读 :ref:`mt_use_case` 指南，了解构建此类应用程序的详细示例。

.. _distributing_by_entity_id:

实时应用
-------------

虽然多租户架构引入了分层结构并使用数据协同定位来为每个租户路由查询，但实时架构依赖于其数据的特定分布属性来实现高度并行处理。

我们使用“实体ID”作为实时模型中分布列的术语，而不是多租户模型中的租户ID。典型实体是用户，主机或设备。

实时查询通常会要求按日期或类别分组的数字聚合。Citus将这些查询发送到每个分片以获得部分结果，并在协调者节点上汇总最终答案。查询在尽可能多的节点参与，并且没有一个节点必须做过多的工作时运行得最快。

最佳实践
^^^^^^^^^^^^^

* **选择具有高基数的列作为分布列。**
  为了进行比较，订单表上的“status”字段具有值"new," "paid," 和 "shipped"，这是分布列的不良选择，因为它仅假设这些值很少。不同值的数量限制了可以容纳数据的分片数量，以及可以处理数据的节点数量。在具有高基数的列中，还可以选择在group-by子句中常用的那些或作为连接键的那些。

* **选择均匀分布的列。**
  如果在偏向某些公共值的列上分布表，则表中的数据将倾向于在某些分片中累积。持有这些分片的节点最终会比其他节点做更多的工作。

* **在其公共列上分布** (`事实和维度表 <https://www.jianshu.com/p/e90e580c0fc9>`_)
  您的事实表只能有一个分布键。连接在另一个键上的表将不与事实表共存。根据连接的频率和连接行的大小选择一个维度进行共同定位。

* **将一些维度表更改为引用表。**
  如果维度表不能与事实表共存，则可以通过将维度表的副本分布到 :ref:`引用表 <reference_tables>` 形式的所有节点来提高查询性能。

阅读 :ref:`rt_use_case` 指南，了解构建此类应用程序的详细示例。

.. _distributing_hash_time:

时间序列数据
-----------------

在时间序列工作负载中，应用程序在归档旧信息时查询最新信息。

在Citus中建模时间序列信息时最常见的错误是使用时间戳本身作为分布列。基于时间的散列分布将看似随机的时间分配到不同的分片中，而不是将时间范围保持在分片中。但是，涉及时间的查询通常会引用时间范围（例如最新数据），因此这种散列分布会导致网络开销。

最佳实践
^^^^^^^^

* **不要选择时间戳作为分布列。**
  选择其他分配列。在多租户应用中，使用租户ID，或在实时应用中使用实体ID。
* **使用PostgreSQL表分区代替时间。**
  使用表分区将时间排序数据的大表分成多个继承表，每个表包含不同的时间范围。在Citus中分布Postgres分区表会为继承的表创建分片。

阅读 :ref:`timeseries` 指南，了解构建此类应用程序的详细示例。

.. _colocation:

表共同位置
---------------

由于其巨大的灵活性和可靠性，关系数据库是许多应用程序的数据存储的首选。从历史上看，关系数据库的一个批评是它们只能在一台机器上运行，这在数据存储需要超过服务器改进时会产生固有的局限性。快速扩展数据库的解决方案是分布它们，但这会产生自身的性能问题：关联操作（如连接）需要跨越网络边界。协同定位是一种在战术上划分数据的做法，其中一个人将相关信息保存在同一台机器上以实现高效的关系操作，但利用了整个数据集的水平可扩展性。

数据共址的原则是数据库中的所有表都有一个公共分布列，并以相同的方式跨机器进行分片，这样具有相同分配列值的行总是在同一台机器上，甚至跨越不同的表。只要分布列提供有意义的数据分组，就可以在组内执行关系操作。

.. _hash_space:

Citus中用于哈希分布表的数据共址
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

PostgreSQL的Citus扩展在能够形成数据库的分布式数据库方面是独一无二的。Citus集群中的每个节点都是功能齐全的PostgreSQL数据库，Citus在多个数据库顶部添加了单个相同数据库的体验。虽然它不能以分布式方式提供PostgreSQL的全部功能，但在许多情况下，它可以通过协同定位（包括完整的SQL支持，事务和外键）充分利用PostgreSQL在单台机器上提供的功能。

在Citus中，如果分布列中值的哈希值落在分片的哈希范围内，则会将一行存储在分片中。为确保协同定位，即使在重新平衡操作之后，具有相同散列范围的分片也始终放在同一节点上，这样相等的分布列值始终位于表的同一节点上。

.. image:: ../images/colocation-shards.png

我们发现在实践中运行良好的分布列是多租户应用程序中的租户ID。例如，SaaS应用程序通常有许多租户，但他们所做的每个查询都特定于特定租户。虽然一种选择是为每个租户提供数据库或模式，但由于可能存在跨用户的许多操作（数据加载，迁移，聚合，分析，架构更改，备份等），因此通常成本高且不切实际。随着租户数量的增加，这变得越来越难以管理。

共址的实际例子
^^^^^^^^^^^^^^^^^^^^^

请考虑以下表，这些表可能是多租户网站分析SaaS的一部分：

.. code-block:: postgresql

  CREATE TABLE event (
    tenant_id int,
    event_id bigint,
    page_id int,
    payload jsonb,
    primary key (tenant_id, event_id)
  );

  CREATE TABLE page (
    tenant_id int,
    page_id int,
    path text,
    primary key (tenant_id, page_id)
  );

现在，我们想要回答可能由面向客户的仪表板发出的查询，例如："返回在租户6中, 以'/blog'开头的所有页面过去一周的访问次数"。

使用常规PostgreSQL表
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如果我们的数据在一个PostgreSQL节点中，我们可以使用SQL提供的丰富的关系操作来轻松表达我们的查询：

.. code-block:: postgresql

  SELECT page_id, count(event_id)
  FROM
    page
  LEFT JOIN  (
    SELECT * FROM event
    WHERE (payload->>'time')::timestamptz >= now() - interval '1 week'
  ) recent
  USING (tenant_id, page_id)
  WHERE tenant_id = 6 AND path LIKE '/blog%'
  GROUP BY page_id;


只要此查询的 `工作集 <https://en.wikipedia.org/wiki/Working_set>`_ 适合内存，这是适用于许多应用程序的合适解决方案，因为它提供了最大的灵活性。然而，即使您还不需要扩展，考虑扩展对数据模型的影响也是有用的。

按ID分布表
^^^^^^^^^^^^^^^^^

随着租户的数量和为每个租户存储的数据的增长，查询时间通常会随着工作集不再适合内存或CPU成为瓶颈而上升。在这种情况下，我们可以使用Citus在多个节点上对数据进行分片。分片时，我们需要做出的第一个也是最重要的选择是分配列。
让我们从一个简单的选择开始, :code:`event` 表使用 :code:`event_id`，:code:`page` 表使用 :code:`page_id`:

.. code-block:: postgresql

  -- 自然地使用event_id和page_id作为分布列

  SELECT create_distributed_table('event', 'event_id');
  SELECT create_distributed_table('page', 'page_id');

鉴于数据分散在不同的工作者之间，我们不能像在单个PostgreSQL节点上那样简单地执行连接。相反，我们需要发出两个查询：

遍历page表的所有分片（Q1）：

.. code-block:: postgresql

  SELECT page_id FROM page WHERE path LIKE '/blog%' AND tenant_id = 6;

遍历event表的所有分片（Q2）：

.. code-block:: postgresql

  SELECT page_id, count(*) AS count
  FROM event
  WHERE page_id IN (/*…第一个查询中的页面IDs…*/)
    AND tenant_id = 6
    AND (payload->>'time')::date >= now() - interval '1 week'
  GROUP BY page_id ORDER BY count DESC LIMIT 10;

之后，两个步骤的结果需要由应用程序组合。

回答查询所需的数据分散在不同节点上的分片中，并且需要查询每个分片：

.. image:: ../images/colocation-inefficient-queries.png

在这种情况下，数据分布会产生很多缺点：

* 查询每个分片，运行多个查询的开销
* Q1的开销将许多行返回给客户端
* Q2变得非常大
* 需要在多个步骤中编写查询，组合结果，需要在应用程序中进行更改

分散相关数据的潜在好处是可以并行化查询，而Citus将会这样做。但是，如果查询所做的工作量远远大于查询许多分片的开销，这只是有益的。通常最好避免直接从应用程序执行此类繁重操作，例如通过 :ref:`pre-aggregating <rollups>` 数据。

按租户分布表
^^^^^^^^^^^^^^^^^

再次查看我们的查询，我们可以看到查询所需的所有行都有一个共同的维度：:code:`tenant_id`。仪表板只会查询租户自己的数据。这意味着如果同一租户的数据始终位于单个PostgreSQL节点上，那么该节点可以通过执行连接 :code:`tenant_id` 和 :code:`page_id` 而在一个步骤中回答我们的原始查询。

在Citus中，具有相同分布列值的行保证位于同一节点上。分布式表中的每个分片实际上都有一组来自其他分布式表的共同定位的分片，它们包含相同的分布列值(相同租户的数据)。重新开始，我们可以使用 :code:`tenant_id` 分布列创建表。

.. code-block:: postgresql

  -- 通过使用公共分布列来co-locate表
  SELECT create_distributed_table('event', 'tenant_id');
  SELECT create_distributed_table('page', 'tenant_id', colocate_with => 'event');

在这种情况下，Citus可以回答您在没有修改的情况下在单个PostgreSQL节点上运行的相同查询（Q1）：

.. code-block:: postgresql

  SELECT page_id, count(event_id)
  FROM
    page
  LEFT JOIN  (
    SELECT * FROM event
    WHERE (payload->>'time')::timestamptz >= now() - interval '1 week'
  ) recent
  USING (tenant_id, page_id)
  WHERE tenant_id = 6 AND path LIKE '/blog%'
  GROUP BY page_id;

由于tenantid过滤和连接tenantid，Citus知道可以使用包含该特定租户的数据的co-located分片集来回答整个查询，并且PostgreSQL节点可以在一个步骤中回答查询，启用完整的SQL支持。

.. image:: ../images/colocation-better-query.png

在某些情况下，查询和表模式将需要进行少量修改，以确保tenant_id始终包含在唯一约束和连接条件中。然而，这通常是一个简单的修改，并且避免了在没有共址的情况下所需的大量重写。

虽然上面的示例只查询一个节点，因为有一个特定的tenant_id=6过滤器，但co-location也允许我们在所有节点上有效地在tenant_id上执行分布式连接，尽管存在SQL限制。

协同定位意味着更好的功能支持
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

通过共址解锁的Citus功能的完整列表包括：

* 对单个共址分片集上的查询的完整SQL支持
* 多语句事务支持对单组共址分片的修改
* 通过INSERT..SELECT进行聚合
* 外键
* 分布式外连接

数据共址是一种强大的技术，可以为关系数据模型提供横向扩展和支持。使用通过共址实现关系操作的分布式数据库迁移或构建应用程序的成本通常远低于转移到限制性数据模型（例如NoSQL），并且与单节点数据库不同，它可以按比例扩展你的业务有关迁移现有数据库的详细信息，请参阅 :ref:`转换为多租户数据模型 <transitioning_mt>`。

.. _query_performance:

查询性能
^^^^^^^^^^^

Citus通过将传入查询分解为多个片段查询（“任务”）来并行处理，这些查询在工作分片上并行运行。这使Citus能够利用集群中所有节点的处理能力以及每个查询的每个节点上的各个核心的处理能力。由于这种并行化，您可以获得性能，该性能是集群中所有核心的计算能力的累积，导致与单个服务器上的PostgreSQL相比查询时间显着减少。

在规划SQL查询时，Citus使用两阶段优化器。第一阶段涉及将SQL查询转换为其可交换和关联形式，以便可以向下推送它们并在并行运行工作程序。如前面部分所述，选择正确的分布列和分布方法允许分布式查询计划程序对查询应用多个优化。由于网络I / O减少，这会对查询性能产生重大影响。

Citus的分布式执行程序然后获取这些单独的查询片段并将它们发送到工作者PostgreSQL实例。分布式规划器和执行器都有几个方面可以调整以提高性能。当这些单独的查询片段被发送给工作者时，查询优化的第二阶段就会启动。工作人员只是运行扩展的PostgreSQL服务器，他们应用PostgreSQL的标准规划和执行逻辑来运行这些片段SQL查询。因此，任何有助于PostgreSQL的优化也有助于Citus。PostgreSQL默认配置保守的资源设置; 因此优化这些配置设置可以显着缩短查询时间。

我们将在文档的 :ref:`performance_tuning` 部分中讨论相关的性能调优步骤。
