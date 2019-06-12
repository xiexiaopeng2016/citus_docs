.. _dml:

摄取，修改数据（DML）
===========================

插入数据
------------

要将数据插入分布式表，可以使用标准的PostgreSQL `INSERT <http://www.postgresql.org/docs/current/static/sql-insert.html>`_ 命令。例如，我们从Github Archive数据集中随机选取两行。

.. code-block:: sql

    /*
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
    */

    INSERT INTO github_events VALUES (2489373118,'PublicEvent','t',24509048,'{}','{"id": 24509048, "url": "https://api.github.com/repos/SabinaS/csee6868", "name": "SabinaS/csee6868"}','{"id": 2955009, "url": "https://api.github.com/users/SabinaS", "login": "SabinaS", "avatar_url": "https://avatars.githubusercontent.com/u/2955009?", "gravatar_id": ""}',NULL,'2015-01-01 00:09:13');

    INSERT INTO github_events VALUES (2489368389,'WatchEvent','t',28229924,'{"action": "started"}','{"id": 28229924, "url": "https://api.github.com/repos/inf0rmer/blanket", "name": "inf0rmer/blanket"}','{"id": 1405427, "url": "https://api.github.com/users/tategakibunko", "login": "tategakibunko", "avatar_url": "https://avatars.githubusercontent.com/u/1405427?", "gravatar_id": ""}',NULL,'2015-01-01 00:00:24');

将行插入分布式表时，必须指定要插入的行的分布列。根据分布列，Citus确定应将插入路由到的正确分片。然后，查询将转发到正确的分片，并在该分片的所有副本上执行远程插入命令。

有时将多个插入语句放在一起插入多行的单个插入是很方便的。它也比重复数据库查询更有效。例如，上一节中的示例可以像这样一次加载：

.. code-block:: sql

    INSERT INTO github_events VALUES
      (
        2489373118,'PublicEvent','t',24509048,'{}','{"id": 24509048, "url": "https://api.github.com/repos/SabinaS/csee6868", "name": "SabinaS/csee6868"}','{"id": 2955009, "url": "https://api.github.com/users/SabinaS", "login": "SabinaS", "avatar_url": "https://avatars.githubusercontent.com/u/2955009?", "gravatar_id": ""}',NULL,'2015-01-01 00:09:13'
      ), (
        2489368389,'WatchEvent','t',28229924,'{"action": "started"}','{"id": 28229924, "url": "https://api.github.com/repos/inf0rmer/blanket", "name": "inf0rmer/blanket"}','{"id": 1405427, "url": "https://api.github.com/users/tategakibunko", "login": "tategakibunko", "avatar_url": "https://avatars.githubusercontent.com/u/1405427?", "gravatar_id": ""}',NULL,'2015-01-01 00:00:24'
      );

"From Select"子句（分布式汇总）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Citus还支持 ``INSERT … SELECT`` 语句 - 根据选择查询的结果插入行。这是一种填充表的便捷方式，也允许使用 `ON CONFLICT <https://yq.aliyun.com/articles/74419>`_ 子句"upserts"，这是执行分布式汇总的最简单方法。

在Citus中，有两种方法可以从select语句中插入。第一种是如果源表和目标表是 :ref:`共址 <colocation>`，并且select/insert语句都包括分布列。在这种情况下，Citus可以将 ``INSERT … SELECT`` 语句推送到所有节点上并行执行。

执行 ``INSERT … SELECT`` 语句的第二种方法是从工作者节点中选择结果，将数据拉到协调者节点，然后从协调者发出带有数据的INSERT语句。当源表和目标表没有共址时，Citus被迫使用这种方法。由于网络开销，此方法效率不高。

如果upserts是应用程序中的一项重要操作，那么理想的解决方案是对数据进行建模，以便对源表和目标表进行共址，并使分布列可以成为upsert语句中GROUP BY子句的一部分(如果进行聚合)。这允许操作在工作者节点之间并行运行，以获得最大速度。

如果对Citus使用的方法有疑问，请使用EXPLAIN命令，如 :ref:`postgresql_tuning` 中所述。

COPY命令（批量加载）
~~~~~~~~~~~~~~~~~~~~~~~~~~~

要批量加载文件中的数据，可以直接使用 PostgreSQL的 \\ `COPY命令 <http://www.postgresql.org/docs/current/static/app-psql.html#APP-PSQL-META-COMMANDS-COPY>`_。

首先通过运行以下命令下载我们的示例github_events数据集：

.. code-block:: bash

    wget http://examples.citusdata.com/github_archive/github_events-2015-01-01-{0..5}.csv.gz
    gzip -d github_events-2015-01-01-*.gz

然后，您可以使用psql复制数据：

.. code-block:: psql

    \COPY github_events FROM 'github_events-2015-01-01-0.csv' WITH (format CSV)

.. note::

    跨分片没有快照隔离的概念，这意味着与COPY同时运行的多分片SELECT语句可能会在某些分片上看到它的提交，但在其他分片上却没有。如果用户正在存储事件数据，他可能偶尔会观察到最近数据中的小间隙。如果这是一个问题，则由应用程序来处理(例如，从查询中排除最近的数据，或使用一些锁)。

    如果COPY语句连接分片位置失败，则其行为方式与INSERT相同，即将位置标记为非活动状态，除非没有更多活动位置。如果在连接后发生任何其他故障，则回滚事务，因此不会进行元数据更改。

.. _rollups:

使用汇总缓存聚合
=====================

事件数据管道和实时仪表板等应用程序需要对大量数据进行亚秒级查询。快速进行这些查询的一种方法是提前计算和保存聚合。这称为“卷起”数据，它避免了在运行时处理原始数据的成本。作为额外的好处，将时间序列数据汇总为每小时或每日统计数据也可以节省空间。当不再需要完整的详细信息并且聚合就足够时，可能会删除旧数据。

例如，这是一个用于通过url跟踪页面视图的分布式表：

.. code-block:: postgresql

  CREATE TABLE page_views (
    site_id int,
    url text,
    host_ip inet,
    view_time timestamp default now(),

    PRIMARY KEY (site_id, url)
  );

  SELECT create_distributed_table('page_views', 'site_id');

一旦表中填充了数据，我们就可以运行聚合查询来计算每天每个URL的页面访问量，并将其限制在给定的站点和年份。

.. code-block:: postgresql

  -- 网站5每天每个网址的访问次数是多少？
  SELECT view_time::date AS day, site_id, url, count(*) AS view_count
    FROM page_views
    WHERE site_id = 5 AND
      view_time >= date '2016-01-01' AND view_time < date '2017-01-01'
    GROUP BY view_time::date, site_id, url;

上面描述的设置可以工作，但是有两个缺点。首先，当您重复执行聚合查询时，它必须遍历每个相关行并重新计算整个数据集的结果。如果您使用此查询呈现仪表板，则可以将聚合结果保存在每日页面视图表中并查询该表会更快。
其次，存储成本将与数据量和可查询历史记录的长度成比例增长。实际上，您可能希望在短时间内保留原始事件，并查看较长时间窗口中的历史图表。

为了获得这些好处，我们可以创建一个 :code:`daily_page_views` 表来存储每日统计数据。

.. code-block:: postgresql

  CREATE TABLE daily_page_views (
    site_id int,
    day date,
    url text,
    view_count bigint,
    PRIMARY KEY (site_id, day, url)
  );

  SELECT create_distributed_table('daily_page_views', 'site_id');

在这个例子中，我们都在 :code:`site_id` 列上分布 :code:`page_views` 和 :code:`daily_page_views`。这确保了与特定站点相对应的数据将 :ref:`co-located <colocation>` 同一节点上。在每个节点上将两个表的行保持在一起可以最大限度地减少节点之间的网络流量，并实现高度并行执行。

一旦我们创建了这个新的分布式表，我们就可以运行 :code:`INSERT INTO ... SELECT` 将原始页面视图汇总到聚合表中。在下文中，我们每天聚合页面视图。Citus用户经常在一天结束后等待一段时间来运行这样的查询，以适应迟到的数据。

.. code-block:: postgresql

  -- roll up yesterday's data
  INSERT INTO daily_page_views (day, site_id, url, view_count)
    SELECT view_time::date AS day, site_id, url, count(*) AS view_count
    FROM page_views
    WHERE view_time >= date '2017-01-01' AND view_time < date '2017-01-02'
    GROUP BY view_time::date, site_id, url;

  -- now the results are available right out of the table
  SELECT day, site_id, url, view_count
    FROM daily_page_views
    WHERE site_id = 5 AND
      day >= date '2016-01-01' AND day < date '2017-01-01';

上面的汇总查询聚合了前一天的数据并将其插入 :code:`daily_page_views`。每天运行一次查询意味着不需要更新汇总表行，因为新日期的数据不会影响以前的行。

处理延迟到达的数据或每天多次运行汇总查询时，情况会发生变化。如果任何新行与汇总表中已有的日期匹配，则匹配计数应该增加。PostgreSQL可以通过“ON CONFLICT”处理这种情况，这是它用于进行 `upserts <https://www.postgresql.org/docs/current/static/sql-insert.html#SQL-ON-CONFLICT>`_ 的技术。这是一个例子。

.. code-block:: postgresql

  -- 从给定日期开始累积，
  -- 在必要时更新每日页面视图
  INSERT INTO daily_page_views (day, site_id, url, view_count)
    SELECT view_time::date AS day, site_id, url, count(*) AS view_count
    FROM page_views
    WHERE view_time >= date '2017-01-01'
    GROUP BY view_time::date, site_id, url
    ON CONFLICT (day, url, site_id) DO UPDATE SET
      view_count = daily_page_views.view_count + EXCLUDED.view_count;

更新和删除
----------

您可以使用标准PostgreSQL `UPDATE <http://www.postgresql.org/docs/current/static/sql-update.html>`_ 和 `DELETE <http://www.postgresql.org/docs/current/static/sql-delete.html>`_ 命令更新或删除分布式表中的行。

.. code-block:: sql

    DELETE FROM github_events
    WHERE repo_id IN (24509048, 24509049);

    UPDATE github_events
    SET event_public = TRUE
    WHERE (org->>'id')::int = 5430905;

当更新/删除影响多个分片时，如上例所示，Citus默认使用单阶段提交协议。为了更加安全，您可以通过设置启用两阶段提交

.. code-block:: postgresql

  SET citus.multi_shard_commit_protocol = '2pc';

如果更新或删除仅影响单个分片，则它将在单个工作者节点内运行。在这种情况下，不需要启用2PC。当更新或删除按表的分布列过滤时，通常会发生这种情况：

.. code-block:: postgresql

  -- 由于github_events由repo_id分发，
  -- 这将在单个工作者节点中执行

  DELETE FROM github_events
  WHERE repo_id = 206084;

此外，在处理单个分片时，Citus支持 ``SELECT … FOR UPDATE``。这是一种有时由对象关系映射器（ORM）用于安全的技术：

1. 加载行
2. 在应用程序代码中进行计算
3. 根据计算更新行

选择要更新的行会对它们设置写入锁，以防止其他进程导致“丢失的更新”异常。

.. code-block:: sql

  BEGIN;

    -- select events for a repo, but
    -- lock them for writing
    SELECT *
    FROM github_events
    WHERE repo_id = 206084
    FOR UPDATE;

    -- calculate a desired value event_public using
    -- application logic that uses those rows...

    -- now make the update
    UPDATE github_events
    SET event_public = :our_new_value
    WHERE repo_id = 206084;

  COMMIT;

这个特性只支持散列分布表和引用表，并且只支持那些 :ref:`replication_factor <replication_factor>` 为1的表。

最大化写入性能
-------------------

在大型计算机上，INSERT和UPDATE/DELETE语句都可以扩展到每秒大约50,000个查询。但是，要实现此速率，您需要使用许多并行，长连接并考虑如何处理锁定。有关更多信息，请参阅我们文档中的 :ref:`scaling_data_ingestion` 部分。
