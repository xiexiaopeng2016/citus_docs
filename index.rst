Citus文档
=========

欢迎使用Citus 8.2的文档！Citus使用分片和复制在商品服务器上横向扩展PostgreSQL。它的查询引擎将这些服务器上的传入SQL查询并行化，以实现对大型数据集的实时响应。

.. raw:: html

  <div class="front-menu row">
    <div class="col-md-4">
      <a class="box" href="portals/getting_started.html">
        <h3>入门</h3>
        <img src="_images/number-one.png" />
        了解Citus体系结构，在本地安装，并遵循一些十分钟的教程。
      </a>
    </div>
    <div class="col-md-4">
      <a class="box" href="portals/use_cases.html">
        <h3>用例</h3>
        <img src="_images/use-cases.png" />
        了解Citus如何通过最少的数据库更改来扩展多租户应用程序。
      </a>
    </div>
    <div class="col-md-4">
      <a class="box" href="portals/migrating.html">
        <h3>迁移到Citus</h3>
        <img src="_images/migrating.png" />
        从简单的PostgreSQL转移到Citus，并发现分布式系统的数据建模技术。
      </a>
    </div>
  </div>
  <div class="front-menu row">
    <div class="col-md-4">
      <a class="box" href="portals/citus_cloud.html">
        <h3>Citus Cloud</h3>
        <img src="_images/cloud.png" />
        探索我们安全，可扩展，高度可用的数据库即服务。
      </a>
    </div>
    <div class="col-md-4">
      <a class="box" href="portals/reference.html">
        <h3>API / 参考</h3>
        <img src="_images/reference.png" />
        通过了解Citus的功能和配置，充分利用Citus。
      </a>
    </div>
    <div class="col-md-4">
      <a class="box" href="portals/support.html">
        <h3>帮助和支持</h3>
        <img src="_images/help.png" />
        请参阅常见问题解答，并与我们联系。这是要解开的页面。
      </a>
    </div>
  </div>

.. toctree::
   :caption: 开始
   :hidden:

   get_started/what_is_citus.rst
   get_started/tutorials.rst

.. toctree::
   :caption: 安装
   :hidden:

   installation/single_machine.rst
   installation/multi_machine.rst
   installation/citus_cloud.rst

.. toctree::
   :caption: 用户案例指南
   :hidden:

   use_cases/multi_tenant.rst
   use_cases/realtime_analytics.rst
   use_cases/timeseries.rst

.. toctree::
   :caption: 架构
   :hidden:

   get_started/concepts.rst
   arch/mx.rst

.. toctree::
   :caption: 开发
   :hidden:

   develop/app_type.rst
   sharding/data_modeling.rst
   develop/migration.rst
   develop/reference.rst
   develop/api.rst
   develop/integrations.rst

.. toctree::
   :caption: 管理
   :hidden:

   admin_guide/cluster_management.rst
   admin_guide/table_management.rst
   admin_guide/upgrading_citus.rst

.. toctree::
   :caption: 答疑
   :hidden:

   performance/performance_tuning.rst
   admin_guide/diagnostic_queries.rst
   reference/common_errors.rst

.. toctree::
   :caption: FAQ
   :hidden:

   faq/faq.rst

.. toctree::
   :caption: 文章
   :hidden:

   articles/index.rst

.. Declare these images as dependencies so that
.. sphinx copies them. It can't detect them in
.. the embedded raw html

.. image:: images/icons/number-one.png
  :width: 0%
.. image:: images/icons/use-cases.png
  :width: 0%
.. image:: images/icons/migrating.png
  :width: 0%
.. image:: images/icons/cloud.png
  :width: 0%
.. image:: images/icons/reference.png
  :width: 0%
.. image:: images/icons/help.png
  :width: 0%
.. image:: images/logo.png
  :width: 0%
.. image:: images/cloud-bill-credit.png
  :width: 0%
.. image:: images/cloud-bill-ach.png
  :width: 0%
