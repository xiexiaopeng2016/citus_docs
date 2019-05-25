.. _what_is_citus:

什么是Citus？
==============

Citus基本上是 `无忧无虑的Postgres<https://www.citusdata.com/product>`_，旨在向外扩展。它是Postgres的扩展，它在多台机器的集群中:ref:`分发数据 <distributed_arch>`和查询。作为扩展（而不是分支），Citus支持新的PostgreSQL版本，允许用户从新功能中受益，同时保持与现有PostgreSQL工具的兼容性。

Citus使用分片和复制在多台机器上横向扩展PostgreSQL。它的查询引擎将这些服务器上的传入并行化SQL查询，以便在大型数据集上实现人性化的实时（不到一秒）的响应。

**有三种使用方式:**

1. 作为 `开源 <https://www.citusdata.com/product/community>`_ 添加到现有的Postgres服务器
2. 内部部署与其他 `企业级 <https://www.citusdata.com/product/enterprise>`_ 安全和集群管理工具
3. 作为Microsoft Azure和Amazon AWS上的服务的完全托管数据库。有关这些产品的信息，请参阅 `Azure Database for PostgreSQL — Hyperscale (Citus) <https://docs.microsoft.com/azure/postgresql/>`_ and `Citus Cloud on AWS <https://www.citusdata.com/product/cloud>`_

.. _how_big:

Citus可以扩展多远？
------------------------

Citus通过添加工作节点水平扩展，通过升级工作者/协调器垂直扩展，并支持切换到 :ref:`mx` 模式（如果需要）. 在实践中，我们的客户已达到以下规模，还有更大的发展空间：

* `Algolia <https://www.citusdata.com/customers/algolia>`_
    * 每天摄入5-10B行
* `Heap <https://www.citusdata.com/customers/heap>`_
    * 500多亿次活动
    * 在40节点Citus数据库集群上有900TB数据
* `Chartbeat <https://www.citusdata.com/customers/chartbeat>`_
    * 每月增加2.6B行以上的数据
* `Pex <https://www.citusdata.com/customers/pex>`_
    * 更新30B行/天
    * Google Cloud上有20节点Citus数据库群集
    * 2.4TB内存，1280个内核和60TB数据
    * 计划增长到45个节点 ...
* `Mixrank <https://www.citusdata.com/customers/mixrank>`_
    * 1.6PB的时间序列数据

更多的客户和统计，请看我们的 `客户案例 <https://www.citusdata.com/customers#customer-index>`_.

.. _when_to_use_citus:

何时使用 Citus
==============

.. _mt_blurb:

多租户数据库
-----------

大多数B2B应用程序已经在其数据模型中内置了租户，客户或帐户的概念。在此模型中，数据库为许多租户提供服务，每个租户的数据与其他租户分开。

Citus为此工作负载提供完整的SQL覆盖，并支持将关系数据库扩展到100K+租户。Citus还为多租户添加了新功能。例如，Citus支持租户隔离，为大型租户提供性能保证，并具有参考表的概念，以减少租户之间的数据重复。

这些功能允许您跨多台计算机扩展租户的数据，并轻松添加更多的CPU，内存和磁盘资源。此外，跨多个租户共享相同的数据库模式可以有效利用硬件资源并简化数据库管理。

Citus对多租户应用的一些优势:

* 快速查询所有租户
* 在数据库中分析逻辑，而不是在应用程序
* 比在单节点PostgreSQL中保存更多的数据
* 在不放弃SQL的情况下向外扩展
* 在高并发性下保持性能
* 跨客户群的快速指标分析
* 轻松扩展以处理新客户注册
* 隔离大小客户的资源使用情况

.. _rt_blurb:

实时分析
-------

Citus支持对大型数据集的实时查询。通常，这些查询发生在快速增长的事件系统或具有时间序列数据的系统中。用例示例包括：

* 具有亚秒响应时间的分析仪表板
* 对展开事件的探索性查询
* 大型数据集归档和报告
* 使用渠道(funnel)，细分(segmentation)和群组(cohort)查询分析会话

Citus的优势在于它能够并行化查询执行并与集群中的工作数据库数量线性扩展。Citus用于实时应用的一些优点：

* 随着数据集的增长，保持亚秒级响应
* 实时分析新事件和新数据
* 并行化SQL查询
* Scale out without giving up SQL
* Maintain performance under high concurrency
* Fast responses to dashboard queries
* Use one database, not a patchwork
* Rich PostgreSQL data types and extensions

Considerations for Use
----------------------

Citus extends PostgreSQL with distributed functionality, but it is not a drop-in replacement that scales out all workloads. A performant Citus cluster involves thinking about the data model, tooling, and choice of SQL features used.

A good way to think about tools and SQL features is the following: if your workload aligns with use-cases described here and you happen to run into an unsupported tool or query, then there’s usually a good workaround.

When Citus is Inappropriate
---------------------------

Some workloads don't need a powerful distributed database, while others require a large flow of information between worker nodes. In the first case Citus is unnecessary, and in the second not generally performant. Here are some examples:

* When single-node Postgres can support your application and you do not expect to grow
* Offline analytics, without the need for real-time ingest nor real-time queries
* Analytics apps that do not need to support a large number of concurrent users
* Queries that return data-heavy ETL results rather than summaries
