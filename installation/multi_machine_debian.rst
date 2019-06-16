.. highlight:: bash

.. _production_deb:

Ubuntu 或 Debian
==================

本节介绍使用deb软件包在您自己的Linux计算机上设置多节点Citus群集所需的步骤。

.. _production_deb_all_nodes:

在所有节点上执行的步骤
---------------------------------

**1. 添加存储库**

::

  # Add Citus repository for package manager
  curl https://install.citusdata.com/community/deb.sh | sudo bash

.. _post_install:

**2. 安装PostgreSQL + Citus并初始化数据库**

::

  # install the server and initialize db
  sudo apt-get -y install postgresql-11-citus-8.2

  # preload citus extension
  sudo pg_conftool 11 main set shared_preload_libraries citus

这将在 `/etc/postgresql/11/main` 中安装集中配置，并在 `/var/lib/postgresql/11/main` 中创建数据库。

.. _post_enterprise_deb:

**3. 配置连接和身份验证**

在启动数据库之前，让我们更改其访问权限。默认情况下，数据库服务器仅侦听localhost上的客户端。作为此步骤的一部分，我们指示它侦听所有IP接口，然后配置客户端身份验证文件以允许来自本地网络的所有传入连接。

::

  sudo pg_conftool 11 main set listen_addresses '*'

::

  sudo vi /etc/postgresql/11/main/pg_hba.conf

::

  # Allow unrestricted access to nodes in the local network. The following ranges
  # correspond to 24, 20, and 16-bit blocks in Private IPv4 address spaces.
  host    all             all             10.0.0.0/8              trust

  # Also allow the host unrestricted access to connect to itself
  host    all             all             127.0.0.1/32            trust
  host    all             all             ::1/128                 trust

.. note::
  您的DNS设置可能有所不同。而且这些设置对于某些环境来说过于宽松，请参阅说明关于 :ref:`worker_security`。PostgreSQL手册 `解释如何 <http://www.postgresql.org/docs/current/static/auth-pg-hba-conf.html>`_ 使它们更具限制性。

**4. 启动数据库服务器，创建Citus扩展**

::

  # start the db server
  sudo service postgresql restart
  # and make it start automatically when computer does
  sudo update-rc.d postgresql enable

您必须将Citus扩展添加到要在群集中使用的 **每个数据库**。以下示例将扩展添加到名为 `postgres` 的默认数据库。

::

  # add the citus extension
  sudo -i -u postgres psql -c "CREATE EXTENSION citus;"

.. _production_deb_coordinator_node:

在协调节点上执行的步骤
--------------------------------------------

在执行前面提到的步骤之后，必须 **仅** 在协调者节点上执行下面列出的步骤。

**1. 添加工作节点信息**

我们需要通知协调者有关其工作者的情况。要添加此信息，我们调用UDF，它将节点信息添加到pg_dist_node登记表中。对于我们的示例，我们假设有两个worker（名为worker-101，worker-102）。将工作者的DNS名称(或IP地址)和服务器端口添加到表中。

::

  sudo -i -u postgres psql -c "SELECT * from master_add_node('worker-101', 5432);"
  sudo -i -u postgres psql -c "SELECT * from master_add_node('worker-102', 5432);"

**2. 验证安装是否成功**

要验证安装是否成功，我们检查协调者节点是否已选择所需的工作者配置。在psql shell中运行此命令应时，应输出我们添加到上面的pg_dist_node表中的工作节点。

::

  sudo -i -u postgres psql -c "SELECT * FROM master_get_active_worker_nodes();"

**准备使用Citus**

在此步骤中，您已完成安装过程并准备好使用Citus群集。可以通过postgres用户在psql中访问新的Citus数据库：

::

  sudo -i -u postgres psql

.. note::

  请注意，Citus会向Citus Data公司服务器报告有关您的群集的匿名信息。要了解有关收集哪些信息以及如何选择退出信息的详细信息，请参阅 :ref:`phone_home`。
