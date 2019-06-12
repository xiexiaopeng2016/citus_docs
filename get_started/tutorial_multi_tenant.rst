.. _multi_tenant_tutorial:

多租户应用
==============

在本教程中，我们将使用示例广告分析数据集来演示如何使用Citus为多租户应用程序提供支持。

.. note::

    本教程假定您已安装并运行Citus。如果您没有运行Citus，您可以：

    * 使用 `Citus Cloud <https://console.citusdata.com/users/sign_up>`_ 配置集群，或

    * 使用 :ref:`single_machine_docker` 在本地设置Citus。


数据模型和样本数据
----------------------

我们将演示为广告分析应用构建数据库，公司可以使用它来查看，更改，分析和管理他们的广告和广告系列（请参阅 `示例应用 <http://citus-example-ad-analytics.herokuapp.com/>`_）。这种应用具有典型的多租户系统的良好特性。来自不同租户的数据存储在中央数据库中，每个租户都有自己数据的独立视图。

我们将使用三个Postgres表来表示这些数据。首先，您需要下载这些表的示例数据：

.. code-block:: bash

    curl https://examples.citusdata.com/tutorial/companies.csv > companies.csv
    curl https://examples.citusdata.com/tutorial/campaigns.csv > campaigns.csv
    curl https://examples.citusdata.com/tutorial/ads.csv > ads.csv

**如果您使用的是Docker**，则应使用该命令 :code:`docker cp` 将文件复制到Docker容器中。

.. code-block:: bash

    docker cp companies.csv citus_master:.
    docker cp campaigns.csv citus_master:.
    docker cp ads.csv citus_master:.

创建表
-----------

首先，您可以使用psql连接到Citus协调者。

**如果您使用的是 Citus Cloud**, 则可以通过指定连接字符串（格式详细信息中的URL）进行连接:

.. code-block:: bash

    psql connection-string

请注意，在连接到Citus Cloud时，某些shell可能要求您引用连接字符串。例如, :code:`psql "connection-string"`.

**如果您使用的是Docker**, 则可以通过使用docker exec命令运行psql来进行连接:

::

    docker exec -it citus_master psql -U postgres

然后，您可以使用标准PostgreSQL命令 :code:`CREATE TABLE` 创建表。

.. code-block:: sql

    CREATE TABLE companies (
        id bigint NOT NULL,
        name text NOT NULL,
        image_url text,
        created_at timestamp without time zone NOT NULL,
        updated_at timestamp without time zone NOT NULL
    );

    CREATE TABLE campaigns (
        id bigint NOT NULL,
        company_id bigint NOT NULL,
        name text NOT NULL,
        cost_model text NOT NULL,
        state text NOT NULL,
        monthly_budget bigint,
        blacklisted_site_urls text[],
        created_at timestamp without time zone NOT NULL,
        updated_at timestamp without time zone NOT NULL
    );

    CREATE TABLE ads (
        id bigint NOT NULL,
        company_id bigint NOT NULL,
        campaign_id bigint NOT NULL,
        name text NOT NULL,
        image_url text,
        target_url text,
        impressions_count bigint DEFAULT 0,
        clicks_count bigint DEFAULT 0,
        created_at timestamp without time zone NOT NULL,
        updated_at timestamp without time zone NOT NULL
    );

接下来，您可以像在PostgreSQL中一样在每个表上创建主键索引

.. code-block:: sql

    ALTER TABLE companies ADD PRIMARY KEY (id);
    ALTER TABLE campaigns ADD PRIMARY KEY (id, company_id);
    ALTER TABLE ads ADD PRIMARY KEY (id, company_id);


分布表和加载数据
--------------------

我们现在继续告诉Citus将这些表分布在集群中的不同节点上。为此，您可以运行 :code:`create_distributed_table` 并指定要进行分片的表和要对其进行分片的列。在这个案例中，我们将用 :code:`company_id` 对所有表进行分片。

.. code-block:: sql

    SELECT create_distributed_table('companies', 'id');
    SELECT create_distributed_table('campaigns', 'company_id');
    SELECT create_distributed_table('ads', 'company_id');

用公司标识符分片的所有表, 允许Citus将表 :ref:`组合 <colocation>` 在一起，并允许像主键，外键和复杂的跨群集连接功能。您可以在 `此处 <https://www.citusdata.com/blog/2016/10/03/designing-your-saas-database-for-high-scalability/>`_ 详细了解此方法的优点。

然后，您可以继续使用标准PostgreSQL :code:`\COPY` 命令将我们下载的数据加载到表中。如果您将文件下载到其他位置，请确保指定正确的文件路径。

.. code-block:: psql

    \copy companies from 'companies.csv' with csv
    \copy campaigns from 'campaigns.csv' with csv
    \copy ads from 'ads.csv' with csv


运行查询
-------------

现在我们已经将数据加载到表中，让我们继续并运行一些查询。Citus支持在分布式表使用标准 :code:`INSERT`, :code:`UPDATE` 和 :code:`DELETE` 命令插入和修改行，这是面向用户的应用程序典型的交互方式。

例如，您可以通过运行以下命令来插入新公司：

.. code-block:: sql

    INSERT INTO companies VALUES (5000, 'New Company', 'https://randomurl/image.png', now(), now());

如果要将公司所有广告系列的预算加倍，可以运行一条UPDATE命令：

.. code-block:: sql

    UPDATE campaigns
    SET monthly_budget = monthly_budget*2
    WHERE company_id = 5;

此类操作的另一个示例是运行跨多个表的事务。假设您要删除广告系列及其所有相关广告，您可以通过运行原子方式删除广告系列。

.. code-block:: sql

    BEGIN;
    DELETE from campaigns where id = 46 AND company_id = 5;
    DELETE from ads where campaign_id = 46 AND company_id = 5;
    COMMIT;

除了事务操作之外，您还可以使用标准SQL对此数据运行分析查询。对一家公司来说，一个有趣的查询是，它需要查看有关其最大预算活动的详细信息。

.. code-block:: sql

    SELECT name, cost_model, state, monthly_budget
    FROM campaigns
    WHERE company_id = 5
    ORDER BY monthly_budget DESC
    LIMIT 10;

我们还可以跨多个表运行join查询，以查看有关正在投放获得最多点击次数和展示次数的广告系列的信息。

.. code-block:: sql

    SELECT campaigns.id, campaigns.name, campaigns.monthly_budget,
           sum(impressions_count) as total_impressions, sum(clicks_count) as total_clicks
    FROM ads, campaigns
    WHERE ads.company_id = campaigns.company_id
    AND campaigns.company_id = 5
    AND campaigns.state = 'running'
    GROUP BY campaigns.id, campaigns.name, campaigns.monthly_budget
    ORDER BY total_impressions, total_clicks;

有了这个，我们将使用Citus为简单的多租户应用程序提供支持的教程结束。下一步，您可以查看多租户应用程序部分，了解如何为多租户建模自己的数据。
