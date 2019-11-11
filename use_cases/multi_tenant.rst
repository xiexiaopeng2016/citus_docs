.. highlight:: postgresql

.. _mt_use_case:

多租户应用程序
=========================

.. contents::

*预计阅读时间: 30分钟*

如果您正在构建软件即服务(SaaS)应用程序，那么您可能已经在数据模型中内置了租户概念。通常，大多数信息涉及租户/客户/帐户，数据库表捕获这种自然关系。

对于SaaS应用程序，每个租户的数据可以一起存储在单个数据库实例中，并与其他租户隔离和不可见。这在三个方面是有效的。首次应用程序改进适用于所有客户。其次，在租户之间共享数据库有效地使用硬件。最后，管理所有租户的单个数据库比为每个租户管理不同的数据库服务器要简单得多。

但是，单个关系数据库实例传统上难以扩展到大型多租户应用程序所需的数据量。当数据超出单个数据库节点的容量时，开发人员被迫放弃关系模型的好处。

Citus允许用户编写多租户应用程序，就像它们连接到单个PostgreSQL数据库一样，而实际上数据库是一个可水平扩展的机器集群。客户端代码只需要很少的修改，并可以继续使用完整的SQL功能。

本指南介绍了多租户应用程序示例，并介绍了如何使用Citus对其进行可扩展性建模。在此过程中，我们将研究多租户应用程序的典型挑战，例如将租户与嘈杂的邻居隔离，扩展硬件以容纳更多数据，以及存储不同租户的数据。PostgreSQL和Citus提供了应对这些挑战所需的所有工具，所以让我们来构建。

让我们制作应用程序 - 广告分析
--------------------------------

我们将为追踪在线广告效果的应用构建后端，并在顶部提供分析仪表板。它非常适合多租户应用程序，因为用户对数据的请求一次只涉及一个(他们自己的)公司。Github上 `提供 <https://github.com/citusdata/citus-example-ad-analytics>`_ 了完整示例应用程序的代码。

让我们首先考虑这个应用程序的简化模式。该应用程序必须跟踪多个公司，每个公司都运行广告活动。广告活动包含许多广告，每个广告都有相关的点击次数和展示次数记录。

这是示例模式。稍后我们将进行一些小的更改，这使我们能够在分布式环境中有效地分发和隔离数据。

::

  CREATE TABLE companies (
    id bigserial PRIMARY KEY,
    name text NOT NULL,
    image_url text,
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL
  );

  CREATE TABLE campaigns (
    id bigserial PRIMARY KEY,
    company_id bigint REFERENCES companies (id),
    name text NOT NULL,
    cost_model text NOT NULL,
    state text NOT NULL,
    monthly_budget bigint,
    blacklisted_site_urls text[],
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL
  );

  CREATE TABLE ads (
    id bigserial PRIMARY KEY,
    campaign_id bigint REFERENCES campaigns (id),
    name text NOT NULL,
    image_url text,
    target_url text,
    impressions_count bigint DEFAULT 0,
    clicks_count bigint DEFAULT 0,
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL
  );

  CREATE TABLE clicks (
    id bigserial PRIMARY KEY,
    ad_id bigint REFERENCES ads (id),
    clicked_at timestamp without time zone NOT NULL,
    site_url text NOT NULL,
    cost_per_click_usd numeric(20,10),
    user_ip inet NOT NULL,
    user_data jsonb NOT NULL
  );

  CREATE TABLE impressions (
    id bigserial PRIMARY KEY,
    ad_id bigint REFERENCES ads (id),
    seen_at timestamp without time zone NOT NULL,
    site_url text NOT NULL,
    cost_per_impression_usd numeric(20,10),
    user_ip inet NOT NULL,
    user_data jsonb NOT NULL
  );

我们可以对模式进行一些修改，这将在Citus这样的分布式环境中提高它的性能。要了解这一点，我们必须熟悉Citus如何分发数据和执行查询。

扩展关系数据模型
---------------------------------

关系数据模型非常适合应用程序。它可以保护数据完整性，允许灵活的查询，并适应不断变化的数据。传统上唯一的问题是关系数据库不被认为能够扩展到大型SaaS应用程序所需的工作负载。
开发人员必须忍受NoSQL数据库 - 或一个后端服务的集合 - 才能达到这个规模。

使用Citus，您可以保留数据模型 *并* 使其扩展。Citus在应用程序中呈现为单个PostgreSQL数据库，但它在内部将查询路由到可调节数量的物理服务器(节点)，这些服务器可以并行处理请求。

多租户应用程序有一个很好的属性，我们可以利用：查询通常总是一次请求一个租户的信息，而不是租户的混合。例如，当销售人员在CRM中搜索潜在客户信息时，搜索结果特定于其雇主; 其他企业的线索和单据不包括在内。

由于应用程序查询仅限于单个租户(例如商店或公司)，因此快速进行多租户应用程序查询的一种方法是将给定租户的 *所有* 数据存储在同一节点上。这可以最大限度地减少节点之间的网络开销，并允许Citus有效地支持所有应用程序的连接，键约束和事务。有了这个，您可以跨多个节点进行扩展，而无需完全重写或重构您的应用程序。

.. image:: ../images/mt-ad-routing-diagram.png


我们通过确保模式中的每个表都有一列来清楚地标记哪个租户拥有哪些行，从而在Citus中执行此操作。在广告分析应用程序中，租户是公司，因此我们必须确保所有表都有一列 :code:`company_id`。

当行被标记为同一公司时，我们可以告诉Citus使用此列来读取和写入行到同一节点。在Citus的术语中 :code:`company_id` 是 *分布列*，您可以在 :ref:`分布式数据建模 <distributed_data_modeling>` 中了解更多信息。

准备表和摄取数据
-----------------------------------

在上一节中，我们为多租户应用程序确定了正确的分布列：公司ID。即使在单机数据库中，通过添加公司ID来对表进行非规范化也是有用的，无论是用于行级安全还是用于附加索引。正如我们所看到的，额外的好处是包括额外的列也有助于多机器扩展。

到目前为止，我们创建的模式使用单独的 :code:`id` 列作为每个表的主键。Citus要求主键和外键约束包括分布列。
这个需求使得在分布式环境中执行这些约束更加有效，因为只需要检查一个节点就可以保证这些约束。

在SQL中，此要求转换为通过包含使主键和外键复合company_id。
在SQL中，这个需求转换为生成包含 :code:`company_id` 的复合主键和外键。
这与多租户案例兼容，因为我们真正需要的是确保每个租户的唯一性。

总而言之，下面的一些修改，是为用:code:`company_id`来分布表做好准备。

::

  CREATE TABLE companies (
    id bigserial PRIMARY KEY,
    name text NOT NULL,
    image_url text,
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL
  );

  CREATE TABLE campaigns (
    id bigserial,       -- was: PRIMARY KEY
    company_id bigint REFERENCES companies (id),
    name text NOT NULL,
    cost_model text NOT NULL,
    state text NOT NULL,
    monthly_budget bigint,
    blacklisted_site_urls text[],
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL,
    PRIMARY KEY (company_id, id) -- added
  );

  CREATE TABLE ads (
    id bigserial,       -- was: PRIMARY KEY
    company_id bigint,  -- added
    campaign_id bigint, -- was: REFERENCES campaigns (id)
    name text NOT NULL,
    image_url text,
    target_url text,
    impressions_count bigint DEFAULT 0,
    clicks_count bigint DEFAULT 0,
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone NOT NULL,
    PRIMARY KEY (company_id, id),         -- added
    FOREIGN KEY (company_id, campaign_id) -- added
      REFERENCES campaigns (company_id, id)
  );

  CREATE TABLE clicks (
    id bigserial,        -- was: PRIMARY KEY
    company_id bigint,   -- added
    ad_id bigint,        -- was: REFERENCES ads (id),
    clicked_at timestamp without time zone NOT NULL,
    site_url text NOT NULL,
    cost_per_click_usd numeric(20,10),
    user_ip inet NOT NULL,
    user_data jsonb NOT NULL,
    PRIMARY KEY (company_id, id),      -- added
    FOREIGN KEY (company_id, ad_id)    -- added
      REFERENCES ads (company_id, id)
  );

  CREATE TABLE impressions (
    id bigserial,         -- was: PRIMARY KEY
    company_id bigint,    -- added
    ad_id bigint,         -- was: REFERENCES ads (id),
    seen_at timestamp without time zone NOT NULL,
    site_url text NOT NULL,
    cost_per_impression_usd numeric(20,10),
    user_ip inet NOT NULL,
    user_data jsonb NOT NULL,
    PRIMARY KEY (company_id, id),       -- added
    FOREIGN KEY (company_id, ad_id)     -- added
      REFERENCES ads (company_id, id)
  );

您可以在 :ref:`多租户架构迁移 <mt_schema_migration>` 中了解有关迁移自己的数据模型的更多信息。

自己试试
~~~~~~~~~~~~~~~

.. note::

  本指南的设计目的是让您能够在自己的Citus数据库中继续学习。使用这些选项之一来启动数据库:

  * 使用 :ref:`single_machine_docker` 在本地运行Citus ，或
  * `注册 <https://console.citusdata.com/users/sign_up>`_ Citus Cloud并配置群集。

  您将使用psql运行SQL命令：

  * **Docker**: :code:`docker exec -it citus_master psql -U postgres`
  * **Cloud**: :code:`psql "connection-string"` 在云控制台中可用的连接字符串。

  在任何一种情况下，psql都将连接到群集中的协调者节点。

此时，您可以通过 `下载 <https://examples.citusdata.com/mt_ref_arch/schema.sql>`_ 并执行SQL来创建模式，自由地在自己的Citus集群中进行操作。模式准备就绪后，我们可以告诉Citus在工作者上创建分片。从协调者节点，运行：

::

  SELECT create_distributed_table('companies',   'id');
  SELECT create_distributed_table('campaigns',   'company_id');
  SELECT create_distributed_table('ads',         'company_id');
  SELECT create_distributed_table('clicks',      'company_id');
  SELECT create_distributed_table('impressions', 'company_id');

该 :ref:`create_distributed_table` 函数通知Citus, 一个表应该节点之间分布, 以及将来这些表的输入查询应规划为分布式执行。该函数还为工作节点上的表创建分片，工作节点是Citus用于向节点分配数据的低级数据存储单元。

下一步是从命令行将样本数据加载到集群中。

.. code-block:: bash

  # download and ingest datasets from the shell

  for dataset in companies campaigns ads clicks impressions geo_ips; do
    curl -O https://examples.citusdata.com/mt_ref_arch/${dataset}.csv
  done

.. note::

  **如果您使用的是Docker,** 则应使用 :code:`docker cp` 命令将文件复制到Docker容器中。

  .. code-block:: bash

    for dataset in companies campaigns ads clicks impressions geo_ips; do
      docker cp ${dataset}.csv citus_master:.
    done

作为PostgreSQL的扩展，Citus支持使用COPY命令进行批量加载。使用它来摄取您下载的数据，并确保在将文件下载到其他位置时指定了正确的文件路径。回到psql里面运行这个：

.. code-block:: psql

  \copy companies from 'companies.csv' with csv
  \copy campaigns from 'campaigns.csv' with csv
  \copy ads from 'ads.csv' with csv
  \copy clicks from 'clicks.csv' with csv
  \copy impressions from 'impressions.csv' with csv

集成应用程序
------------------------

这是一个好消息：一旦你做了前面描述的轻微架构修改，你的应用程序可以用很少的工作进行扩展。您只需将应用程序连接到Citus，让数据库负责保持查询的快速性和数据的安全性。

任何包含 :code:`company_id` 过滤的应用程序查询或更新语句都将继续正常工作。
如前所述，这种过滤在多租户应用程序中很常见。使用对象关系映射(ORM)时，您可以通过诸如 :code:`where` 或 :code:`filter` 之类的方法识别这些查询。

ActiveRecord:

.. code-block:: ruby

  Impression.where(company_id: 5).count

Django:

.. code-block:: py

  Impression.objects.filter(company_id=5).count()

基本上，当在数据库中执行的最终SQL在每个表上包含一个 :code:`WHERE company_id = :value` 子句(包括JOIN查询中的表)时，Citus将认识到该查询应该路由到单个节点并按原样执行。这可确保所有SQL功能都可用。毕竟节点是一个普通的PostgreSQL服务器。

此外，为了使其更简单，您可以使用我们的 `activerecord-multi-tenant <https://github.com/citusdata/activerecord-multi-tenant>`_ Rails库，
或Django的 `django-multitenant <https://github.com/citusdata/django-multitenant>`_ ，它会自动将这些过滤器添加到您的所有查询，甚至是复杂的查询。查看我们的Ruby on Rails和Django迁移指南。

本指南与框架无关，因此我们将使用SQL指出一些Citus功能。利用您的想象力，如何用您选择的语言表达这些陈述。

这是在单个租户上的一个简单的查询和更新操作。

.. code-block:: sql

  -- campaigns with highest budget

  SELECT name, cost_model, state, monthly_budget
    FROM campaigns
   WHERE company_id = 5
   ORDER BY monthly_budget DESC
   LIMIT 10;

  -- double the budgets!

  UPDATE campaigns
     SET monthly_budget = monthly_budget*2
   WHERE company_id = 5;

用户使用NoSQL数据库扩展应用程序的一个常见痛点是缺少事务和连接。但是，在Citus中，事务的工作方式正如您所期望的那样:

.. code-block:: sql

  -- transactionally reallocate campaign budget money

  BEGIN;

  UPDATE campaigns
     SET monthly_budget = monthly_budget + 1000
   WHERE company_id = 5
     AND id = 40;

  UPDATE campaigns
     SET monthly_budget = monthly_budget - 1000
   WHERE company_id = 5
     AND id = 41;

  COMMIT;

作为SQL支持的最终演示，我们有一个包含聚合和窗口函数的查询，它在Citus中的工作方式与在PostgreSQL中的工作方式相同。该查询会根据展示次数对每个广告系列中的广告进行排名。

.. code-block:: sql

  SELECT a.campaign_id,
         RANK() OVER (
           PARTITION BY a.campaign_id
           ORDER BY a.campaign_id, count(*) desc
         ), count(*) as n_impressions, a.id
    FROM ads as a
    JOIN impressions as i
      ON i.company_id = a.company_id
     AND i.ad_id      = a.id
   WHERE a.company_id = 5
  GROUP BY a.campaign_id, a.id
  ORDER BY a.campaign_id, n_impressions desc;

简而言之，当查询范围限定为租户时，插入，更新，删除，复杂SQL和事务都按预期工作。

.. _mt_ref_tables:

在租户之间共享数据
----------------------------

到目前为止，所有表都已用 :code:`company_id` 分布，但有时会有所有租户共享的数据，并且特别不属于任何租户。
例如，使用此示例广告平台的所有公司可能希望根据IP地址获取其受众的地理信息。
在单个机器数据库中，这可以通过geo-ip的查找表来完成，如下所示。(真正的表可能会使用PostGIS，但请使用简化示例。)

.. code-block:: sql

  CREATE TABLE geo_ips (
    addrs cidr NOT NULL PRIMARY KEY,
    latlon point NOT NULL
      CHECK (-90  <= latlon[0] AND latlon[0] <= 90 AND
             -180 <= latlon[1] AND latlon[1] <= 180)
  );
  CREATE INDEX ON geo_ips USING gist (addrs inet_ops);

要在分布式设置中有效地使用此表，我们需要找到一种方法来共址 :code:`geo_ips` 表，不仅仅是一个, 而是每个公司的点击量。这样，在查询时不需要产生网络流量。为此, 我们通过在Citus中指定 :code:`geo_ips` 为 :ref:`参考表 <reference_tables>`。

.. code-block:: sql

  -- Make synchronized copies of geo_ips on all workers

  SELECT create_reference_table('geo_ips');

在所有工作节点上复制参考表，Citus会在修改期间自动使它们保持同步。请注意，我们调用 :ref:`create_reference_table <create_reference_table>` 而不是 :code:`create_distributed_table`。

现在，:code:`geo_ips` 已建立为参考表，请使用示例数据加载它：

.. code-block:: psql

  \copy geo_ips from 'geo_ips.csv' with csv

现在，使用此表加入点击可以有效执行。例如，我们可以询问点击广告290的每个人的位置。

.. code-block:: sql

  SELECT c.id, clicked_at, latlon
    FROM geo_ips, clicks c
   WHERE addrs >> c.user_ip
     AND c.company_id = 5
     AND c.ad_id = 290;

模式的在线更改
----------------------------

多租户系统的另一个挑战是保持所有租户的模式同步。任何模式更改都需要始终反映在所有租户中。在Citus中，您可以简单地使用标准PostgreSQL DDL命令来更改表的模式，Citus将使用两阶段提交协议将它们从协调者节点传播到工作者。

例如，这个应用程序中的广告可以使用文本标题。我们可以通过在协调者上发出标准SQL来向表中添加一个列：

.. code-block:: sql

  ALTER TABLE ads
    ADD COLUMN caption text;

这也更新了所有工作者。此命令完成后，Citus群集将接受在新 :code:`caption`  列中读取或写入数据的查询。

有关DDL命令如何通过群集传播的更全面说明，请参阅 :ref:`ddl_prop_support`。

当租户之间的数据不同时
--------------------------------

考虑到所有租户共享一个公共模式和硬件基础设施，我们如何容纳那些希望存储其他人不需要的信息的租户?
例如，使用我们的广告数据库的一个租户应用程序可能希望通过单击存储跟踪cookie信息，而另一个租户可能关心浏览器代理。
传统上，使用多租户共享模式方法的数据库只能创建固定数量的预先分配的"自定义"列，或者拥有外部"扩展表"。
但PostgreSQL提供了一种更简单的非结构化列类型，特别是 `JSONB <https://www.postgresql.org/docs/current/static/datatype-json.html>`_。

请注意，我们的模式已经在 :code:`clicks` 有了一个名为 :code:`user_data`的JSONB字段。每个租户都可以使用它进行灵活存储。

假设公司5在字段中包含了用于跟踪用户是否在移动设备上的信息。该公司可以查询谁点击得更多，是移动用户还是传统用户:

.. code-block:: postgresql

  SELECT
    user_data->>'is_mobile' AS is_mobile,
    count(*) AS count
  FROM clicks
  WHERE company_id = 5
  GROUP BY user_data->>'is_mobile'
  ORDER BY count DESC;

数据库管理员甚至可以创建一个 `部分索引 <https://www.postgresql.org/docs/current/static/indexes-partial.html>`_ 来提高单个租户的查询模式的速度。这是一个改进公司5的移动设备用户点击过滤器：

.. code-block:: postgresql

  CREATE INDEX click_user_data_is_mobile
  ON clicks ((user_data->>'is_mobile'))
  WHERE company_id = 5;

此外，PostgreSQL在JSONB上支持 `GIN 索引 <https://www.postgresql.org/docs/current/static/gin-intro.html>`_。
在JSONB列上创建一个GIN索引将在该JSON文档中的每个键和值上创建一个索引。
这加快了一些 `JSONB操作 <https://www.postgresql.org/docs/current/static/functions-json.html#FUNCTIONS-JSONB-OP-TABLE>`_，比如 :code:`?`, :code:`?|`, 和 :code:`?&`。

.. code-block:: postgresql

  CREATE INDEX click_user_data
  ON clicks USING gin (user_data);

  -- this speeds up queries like, "which clicks have
  -- the is_mobile key present in user_data?"

  SELECT id
    FROM clicks
   WHERE user_data ? 'is_mobile'
     AND company_id = 5;

扩展硬件资源
--------------------------

.. note::

  本节使用仅在 `Citus Cloud <https://www.citusdata.com/product/cloud>`_ 和 `Citus Enterprise <https://www.citusdata.com/product/enterprise>`_ 中提供的功能。另外，请注意，除了"开发计划"之外，这些特性在Citus Cloud的所有计划中都可用。

随着业务的增长或租户希望存储更多数据，应为未来规模设计多租户数据库。Citus可以通过添加新机器轻松扩展，无需进行任何更改或停止应用程序。

能够在Citus群集中重新平衡数据，可以增加数据大小或客户数量，并根据需要提高性能。添加新机器允许您将数据保存在内存中，即使数据比一台机器能够存储的数据大得多。

此外，如果只有少数大租户的数据增加，那么您可以将这些特定租户隔离到单独的节点以获得更好的性能。

要扩展Citus群集，请先向其添加新的工作节点。在Citus Cloud上，您可以使用"设置"选项卡中的滑块，滑动它以添加所需数量的节点。或者，如果您运行自己的Citus安装，则可以使用 :ref:`master_add_node` 手动添加节点。

.. image:: ../images/cloud-nodes-slider.png

添加节点之后，系统中就可以使用它了。不过，目前还没有租户存储在它上面，Citus还不会在上面运行任何查询。要移动现有数据，您可以要求Citus重新平衡数据。此操作在当前活动节点之间移动称为分片的行束，以尝试均衡每个节点上的数据量。

.. code-block:: postgres

  SELECT rebalance_table_shards('companies');

重新平衡保留了 :ref:`colocation`，这意味着我们可以告诉Citus重新平衡companies表，它将接受提示并重新平衡由company_id分发的其他表。
此外，应用程序在分片重新平衡期间不需要经历停机。读取请求无缝地继续，并且只有当它们影响当前正在传输的分片时才会锁定写入。

您可以在此处了解有关分片重新平衡如何工作的更多信息：:ref:`scaling_out`。

处理大租户
------------------------

.. note::

  本节使用仅Citus Cloud和Citus Enterprise中提供的功能。

上一节描述了随着租户数量的增加扩展集群的一种通用方法。然而，用户通常有两个问题。第一个问题是，如果他们最大的租户发展得太大，会发生什么。第二个是在单个工作节点上托管大租户和小租户的性能影响，以及可以采取哪些措施。

关于第一个问题，调查大型SaaS站点的数据发现，随着租户数量的增加，租户数据的大小通常趋向于遵循一个 `Zipfian分布 <https://en.wikipedia.org/wiki/Zipf%27s_law>`_。

.. image:: ../images/zipf.png

例如，在100个租户的数据库中，预计最大的租户占据了大约20％的数据。在大型SaaS公司的一个更现实的例子中，如果有10k个租户，最大的将占据数据的2％左右。即使是10TB的数据，最大的租户也需要200GB，这很容易适合单个节点。

另一个问题是关于大型和小型租户在同一节点上的性能。
标准的分片再平衡将改善总体性能，但它可能会也可能不会改善大型租户和小型租户的混合。重新平衡器只是分配分片以均衡节点上的存储使用情况，而不检查每个分片上分配了哪些租户。

为了改善资源分配并保证租户QoS，将大租户移动到专用节点是值得的。Citus提供了执行此操作的工具。

在我们的例子中，让我们假设我们的老朋友公司id=5非常大。我们可以通过两个步骤隔离此租户的数据。我们将在此处提供命令，您可以参考 :ref:`tenant_isolation` 以了解有关它们的更多信息。

首先，将租户的数据隔离到一个适合移动的包(称为分片)中。CASCADE选项还将此更改应用于其他使用 :code:`company_id`分布的表。

.. code-block:: sql

  SELECT isolate_tenant_to_new_shard(
    'companies', 5, 'CASCADE'
  );

输出的是专用于保存 :code:`company_id=5` 的分片ID：

.. code-block:: text

  ┌─────────────────────────────┐
  │ isolate_tenant_to_new_shard │
  ├─────────────────────────────┤
  │                      102240 │
  └─────────────────────────────┘

接下来，我们将数据通过网络移动到新的专用节点。按照上一节中的描述创建一个新节点。记下其主机名，如云控制台的节点选项卡中所示。

.. code-block:: sql

  -- find the node currently holding the new shard

  SELECT nodename, nodeport
    FROM pg_dist_placement AS placement,
         pg_dist_node AS node
   WHERE placement.groupid = node.groupid
     AND node.noderole = 'primary'
     AND shardid = 102240;

  -- move the shard to your choice of worker (it will also move the
  -- other shards created with the CASCADE option)

  SELECT master_move_shard_placement(
    102240,
    'source_host', source_port,
    'dest_host', dest_port);

您可以通过再次查询 :ref:`pg_dist_placement <placements>` 来确认分片移动。

下一步如何做
---------------------

有了这个，您现在就知道如何使用Citus为您的多租户应用程序提供可扩展性。如果您有现有模式并希望将其迁移到Citus，请参阅 :ref:`多租户转换 <transitioning_mt>`。

要调整前端应用程序，特别是Ruby on Rails或Django，请阅读 :ref:`rails_migration`。最后，尝试使用 :ref:`Citus Cloud <cloud_overview>`，这是管理Citus群集的最简单方法，可提供打折的开发人员计划定价。
