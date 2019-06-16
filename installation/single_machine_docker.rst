.. _single_machine_docker:

Docker (Mac 或 Linux)
===========================

本节介绍如何使用docker-compose在单台计算机上设置Citus群集。

.. note::
   **Docker镜像仅用于开发/测试** ，尚未准备用于生产用途。镜像使用默认连接设置，这是非常宽松的，不适合任何类型的生产设置。在将镜像用于生产之前，应该更新这些内容。PostgreSQL手册 `解释了如何 <http://www.postgresql.org/docs/current/static/auth-pg-hba-conf.html>`_ 使它们更具限制性。

**1. 安装 Docker 社区版和 Docker Compose**

*在Mac上:*

* 安装 `Docker <https://www.docker.com/community-edition#/download>`_.
* 单击应用程序的图标启动Docker。

*在Linux上:*

.. code-block:: bash

  curl -sSL https://get.docker.com/ | sh
  sudo usermod -aG docker $USER && exec sg docker newgrp `id -gn`
  sudo systemctl start docker

  sudo curl -sSL https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
  sudo chmod +x /usr/local/bin/docker-compose

以上版本的Docker Compose足以运行Citus，或者您可以安装 `latest version <https://github.com/docker/compose/releases/latest>`_。

.. _post_install:

**2. 启动Citus群集**

Citus使用Docker Compose来运行和连接容纳数据库协调者节点，工作者和持久数据卷的容器。要创建本地群集，请下载我们的Docker Compose配置文件并运行它

.. code-block:: bash

  curl -L https://raw.githubusercontent.com/citusdata/docker/master/docker-compose.yml > docker-compose.yml
  COMPOSE_PROJECT_NAME=citus docker-compose up -d

第一次启动集群时，它会构建其容器。后续启动只需几秒钟。

.. note::

  如果您已在计算机上运行PostgreSQL，则在启动Docker容器时可能会遇到此错误：

  .. code::

    Error starting userland proxy:
    Bind for 0.0.0.0:5432: unexpected error address already in use

  这是因为"master"(协调者)服务尝试绑定到标准PostgreSQL端口5432。只需为 ``MASTER_EXTERNAL_PORT`` 环境变量选择一个不同的协调者服务端口。例如：

  .. code::

    MASTER_EXTERNAL_PORT=5433 COMPOSE_PROJECT_NAME=citus docker-compose up -d


**3. 验证安装是否成功**

要验证安装是否成功，我们检查协调者节点是否已选择所需的工作者配置。首先在协调者(master)节点上启动psql shell：

.. code-block:: bash

  docker exec -it citus_master psql -U postgres

然后运行此查询：

.. code-block:: postgresql

  SELECT * FROM master_get_active_worker_nodes();

您应该看到每个工作节点的一行，包括节点名称和端口。

启动并运行群集后，您可以访问我们的 :ref:`多租户应用程序 <multi_tenant_tutorial>` 或 :ref:`实时分析 <real_time_analytics_tutorial>` 教程，以便在几分钟内开始使用Citus。

**4. 准备好后关闭群集**

如果要停止docker容器，请使用Docker Compose：

.. code-block:: bash

  COMPOSE_PROJECT_NAME=citus docker-compose down -v

.. note::

  请注意，Citus会向Citus Data公司服务器报告有关您的群集的匿名信息。要了解有关收集哪些信息以及如何选择退出信息的详细信息，请参阅 :ref:`phone_home`。
