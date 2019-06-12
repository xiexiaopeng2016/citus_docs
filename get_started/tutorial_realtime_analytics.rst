.. _real_time_analytics_tutorial:

实时分析
==============

在本教程中，我们将演示如何使用Citus来实时获取事件数据并对该数据运行分析查询。为此，我们将使用示例Github事件数据集。

.. note::

    本教程假定您已安装并运行Citus。如果您没有运行Citus，您可以：

    * 使用 `Citus Cloud <https://console.citusdata.com/users/sign_up>`_ 配置集群，或

    * 使用 :ref:`single_machine_docker` 在本地设置Citus。


数据模型和样本数据
------------------------

我们将演示为实时分析应用程序构建数据库。此应用程序将插入大量事件数据，并对具有亚秒级延迟的数据启用分析查询。在我们的示例中，我们将使用Github事件数据集。此数据集包括Github上的所有公共事件，例如提交，分支，新问题以及对这些问题的评论。

我们将使用两个Postgres表来表示这些数据。首先，您需要下载这些表的示例数据：

.. code-block:: bash

    curl https://examples.citusdata.com/tutorial/users.csv > users.csv
    curl https://examples.citusdata.com/tutorial/events.csv > events.csv

**如果您使用的是Docker**, 则应使用 :code:`docker cp` 命令将文件复制到Docker容器中.

.. code-block:: bash

    docker cp users.csv citus_master:.
    docker cp events.csv citus_master:.

创建表
-------------

首先，您可以使用psql连接到Citus协调者。

**如果您使用的是 Citus Cloud**, 则可以通过指定连接字符串（格式详细信息中的URL）进行连接:

.. code-block:: bash

    psql connection-string

请注意，在连接到Citus Cloud时，某些shell可能要求您引用连接字符串。例如 :code:`psql `connection-string`。

**如果您使用的是Docker**, 则可以通过使用docker exec命令运行psql来进行连接:

.. code-block:: bash

    docker exec -it citus_master psql -U postgres

然后，您可以使用标准PostgreSQL :code:`CREATE TABLE` 命令创建表。

.. code-block:: sql

    CREATE TABLE github_events
    (
        event_id bigint,
        event_type text,
        event_public boolean,
        repo_id bigint,
        payload jsonb,
        repo jsonb,
        user_id bigint,
        org jsonb,
        created_at timestamp
    );

    CREATE TABLE github_users
    (
        user_id bigint,
        url text,
        login text,
        avatar_url text,
        gravatar_id text,
        display_login text
    );

接下来，您可以像在PostgreSQL中一样在事件数据上创建索引。在这个例子中，我们还将为 :code:`jsonb` 字段创建一个 :code:`GIN` 索引, 使查询更快速。

.. code-block:: sql

    CREATE INDEX event_type_index ON github_events (event_type);
    CREATE INDEX payload_index ON github_events USING GIN (payload jsonb_path_ops);

分发表和加载数据
----------------------

我们现在继续告诉Citus将这些表分布在集群中的节点上。为此，您可以运行 :code:`create_distributed_table` 并指定要进行分片的表和要对其进行分片的列。在这个案例中，我们将使用 :code:`user_id` 对所有表进行分片。

.. code-block:: sql

    SELECT create_distributed_table('github_users', 'user_id');
    SELECT create_distributed_table('github_events', 'user_id');

使用用户标识符进行分片的所有表允许Citus将其 :ref:`colocate <colocation>` 在一起，并且允许有效连接和分布式滚动。您可以在 `此处 <https://www.citusdata.com/blog/2016/11/29/event-aggregation-at-scale-with-postgresql/>`_ 详细了解此方法的优点。

然后，您可以继续使用标准PostgreSQL :code:`\COPY` 命令将我们下载的数据加载到表中。如果将文件下载到其他位置，请确保指定正确的文件路径。

.. code-block:: psql

    \copy github_users from 'users.csv' with csv
    \copy github_events from 'events.csv' with csv


运行查询
---------------

现在我们已经将数据加载到表中，让我们继续并运行一些查询。首先，让我们检查一下我们在分布式数据库中拥有多少用户。

.. code-block:: sql

    SELECT count(*) FROM github_users;

现在，让我们分析一下我们数据中的Github推送事件。我们将首先使用每个推送事件中的不同提交数来计算每分钟的提交数。

.. code-block:: sql

    SELECT date_trunc('minute', created_at) AS minute,
           sum((payload->>'distinct_size')::int) AS num_commits
    FROM github_events
    WHERE event_type = 'PushEvent'
    GROUP BY minute
    ORDER BY minute;

我们还有一个用户表。我们还可以轻松地将用户与事件联系起来，并找到创建最多存储库的前十个用户。

.. code-block:: sql

    SELECT login, count(*)
    FROM github_events ge
    JOIN github_users gu
    ON ge.user_id = gu.user_id
    WHERE event_type = 'CreateEvent' AND payload @> '{"ref_type": "repository"}'
    GROUP BY login
    ORDER BY count(*) DESC LIMIT 10;

Citus还支持标准 :code:`INSERT`, :code:`UPDATE` 以及 :code:`DELETE` 命令用于摄取和修改数据。例如，您可以通过运行以下命令来更新用户的登录后显示的名称：

.. code-block:: sql

    UPDATE github_users SET display_login = 'no1youknow' WHERE user_id = 24305673;

有了这个，我们来到教程的最后。下一步，您可以查看:ref:`distributing_by_entity_id` ，了解如何为自己的数据建模并为实时分析应用程序提供支持。
