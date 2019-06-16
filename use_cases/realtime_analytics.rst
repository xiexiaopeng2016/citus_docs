.. highlight:: postgresql

.. _rt_use_case:

实时仪表盘
====================

Citus提供大型数据集的实时查询。我们在Citus常见的一项工作负载涉及为事件数据的实时仪表板提供驱动。

例如，您有可能成为云服务提供商，帮助其他企业监控其HTTP流量。每当您的一个客户端收到HTTP请求时，您的服务就会收到一条日志记录。您希望获取所有这些记录并创建HTTP分析仪表板，为您的客户提供洞察，例如其网站所服务的HTTP错误数量。重要的是，此数据显示尽可能少的延迟，以便您的客户可以修复其网站的问题。仪表板显示历史趋势图也很重要。

或者，也许您正在建立一个广告网络，并希望在其广告系列中向客户展示点击率。在此示例中，延迟也很关键，原始数据量也很高，历史数据和实时数据都很重要。

在本节中，我们将演示如何构建第一个示例的一部分，但是对于第二个用例和许多其他用例，这个体系结构同样可以很好地工作。

数据模型
----------

我们正在处理的数据是不可变的日志数据流。我们将直接插入Citus，但这些数据首先通过像Kafka这样的路由进行路由也很常见。这样做具有通常的优势，并且一旦数据量变得难以管理地高，就可以更容易地预聚合数据。

我们将使用一个简单的模式来摄取HTTP事件数据。以该模式为例，演示了整个体系结构;一个实际的系统可能会使用额外的列。

.. code-block:: sql

  -- this is run on the coordinator

  CREATE TABLE http_request (
    site_id INT,
    ingest_time TIMESTAMPTZ DEFAULT now(),

    url TEXT,
    request_country TEXT,
    ip_address TEXT,

    status_code INT,
    response_time_msec INT
  );

  SELECT create_distributed_table('http_request', 'site_id');

当我们调用 :ref:`create_distributed_table <create_distributed_table>` 时，
我们要求Citus使用 ``site_id`` 列散列分发 ``http_request``。这意味着特定站点的所有数据都将存在于同一个分片中。

UDF使用分片数目的默认配置值。我们建议在群集中 :ref:`使用2-4倍的分片 <faq_choose_shard_count>` 作为CPU核心。使用这么多分片可以在添加新的工作节点后重新平衡群集中的数据。

.. NOTE::

  Citus Cloud使用 `流复制 <https://www.postgresql.org/docs/current/static/warm-standby.html>`_ 来实现高可用性，因此维护分片副本将是多余的。在流复制不可用的任何生产环境中，都应该设置 ``citus.shard_replication_factor`` 为2或更高，用于容错。

有了这个，系统就可以接受数据并提供查询了！在您继续使用本文中的其他命令时，请在后台的 ``psql`` 控制台中运行以下循环。它每隔一两秒生成一次假数据。

.. code-block:: postgres

  DO $$
    BEGIN LOOP
      INSERT INTO http_request (
        site_id, ingest_time, url, request_country,
        ip_address, status_code, response_time_msec
      ) VALUES (
        trunc(random()*32), clock_timestamp(),
        concat('http://example.com/', md5(random()::text)),
        ('{China,India,USA,Indonesia}'::text[])[ceil(random()*4)],
        concat(
          trunc(random()*250 + 2), '.',
          trunc(random()*250 + 2), '.',
          trunc(random()*250 + 2), '.',
          trunc(random()*250 + 2)
        )::inet,
        ('{200,404}'::int[])[ceil(random()*2)],
        5+trunc(random()*150)
      );
      COMMIT;
      PERFORM pg_sleep(random() * 0.25);
    END LOOP;
  END $$;

一旦您摄入数据，您可以运行仪表板查询，例如：

.. code-block:: sql

  SELECT
    site_id,
    date_trunc('minute', ingest_time) as minute,
    COUNT(1) AS request_count,
    SUM(CASE WHEN (status_code between 200 and 299) THEN 1 ELSE 0 END) as success_count,
    SUM(CASE WHEN (status_code between 200 and 299) THEN 0 ELSE 1 END) as error_count,
    SUM(response_time_msec) / COUNT(1) AS average_response_time_msec
  FROM http_request
  WHERE date_trunc('minute', ingest_time) > now() - '5 minutes'::interval
  GROUP BY site_id, minute
  ORDER BY minute ASC;

上述设置可以工作，但有两个缺点:

* 每次需要生成图表时，您的HTTP分析仪表板都必须遍历每一行。例如，如果您的客户对过去一年的趋势感兴趣，那么您的查询将从头开始汇总过去一年的每一行。
* 您的存储成本将与摄取率和可查询历史记录的长度成比例增长。在实践中，您可能希望将原始事件保留较短的时间段(一个月)，并查看较长时间段(年)的历史图表。

汇总
-------

您可以通过将原始数据汇总到预先聚合的表单来克服这两个缺点。在这里，我们将原始数据聚合到一个表中，该表存储1分钟间隔的摘要。在生产系统中，您可能还需要1小时和1天的间隔，这些间隔对应于仪表板中的缩放级别。当用户需要上个月的请求时间时，仪表板可以简单地读取并绘制最近30天的每个值。

.. code-block:: sql

  CREATE TABLE http_request_1min (
    site_id INT,
    ingest_time TIMESTAMPTZ, -- which minute this row represents

    error_count INT,
    success_count INT,
    request_count INT,
    average_response_time_msec INT,
    CHECK (request_count = error_count + success_count),
    CHECK (ingest_time = date_trunc('minute', ingest_time))
  );

  SELECT create_distributed_table('http_request_1min', 'site_id');

  CREATE INDEX http_request_1min_idx ON http_request_1min (site_id, ingest_time);

这看起来很像以前的代码块。最重要的是：它也是在 ``site_id`` 分片, 并为分片数目和复制因子使用相同的默认配置。因为这三个都匹配，所以在 ``http_request`` 分片和 ``http_request_1min`` 分片之间存在1对1的通信，Citus将把匹配的分片放在同一个工作者上。这称为 :ref:`共址 <colocation>`; 它使连接等查询更快，并使汇总成为可能。

.. image:: /images/colocation.png
  :alt: co-location in citus


为了填充 ``http_request_1min``, 我们将定期运行INSERT INTO SELECT。这是可能的，因为表位于同一位置。以下函数为方便起见包装了汇总查询。

.. code-block:: plpgsql

  -- single-row table to store when we rolled up last
  CREATE TABLE latest_rollup (
    minute timestamptz PRIMARY KEY,

    -- "minute" should be no more precise than a minute
    CHECK (minute = date_trunc('minute', minute))
  );

  -- initialize to a time long ago
  INSERT INTO latest_rollup VALUES ('10-10-1901');

  -- function to do the rollup
  CREATE OR REPLACE FUNCTION rollup_http_request() RETURNS void AS $$
  DECLARE
    curr_rollup_time timestamptz := date_trunc('minute', now());
    last_rollup_time timestamptz := minute from latest_rollup;
  BEGIN
    INSERT INTO http_request_1min (
      site_id, ingest_time, request_count,
      success_count, error_count, average_response_time_msec
    ) SELECT
      site_id,
      date_trunc('minute', ingest_time),
      COUNT(1) as request_count,
      SUM(CASE WHEN (status_code between 200 and 299) THEN 1 ELSE 0 END) as success_count,
      SUM(CASE WHEN (status_code between 200 and 299) THEN 0 ELSE 1 END) as error_count,
      SUM(response_time_msec) / COUNT(1) AS average_response_time_msec
    FROM http_request
    -- roll up only data new since last_rollup_time
    WHERE date_trunc('minute', ingest_time) <@
            tstzrange(last_rollup_time, curr_rollup_time, '(]')
    GROUP BY 1, 2;

    -- update the value in latest_rollup so that next time we run the
    -- rollup it will operate on data newer than curr_rollup_time
    UPDATE latest_rollup SET minute = curr_rollup_time;
  END;
  $$ LANGUAGE plpgsql;

.. note::

  应该每分钟调用上述函数。您可以通过在协调节点上添加定时任务来执行此操作：

  .. code-block:: bash

    * * * * * psql -c 'SELECT rollup_http_request();'

  或者，诸如 `pg_cron <https://github.com/citusdata/pg_cron>`_ 之类的扩展允许您直接从数据库安排循环查询。

之前的仪表板查询现在好多了：

.. code-block:: sql

  SELECT site_id, ingest_time as minute, request_count,
         success_count, error_count, average_response_time_msec
    FROM http_request_1min
   WHERE ingest_time > date_trunc('minute', now()) - '5 minutes'::interval;

过期旧数据
-----------------

汇总使查询更快，但我们仍需要使旧数据过期以避免无限的存储成本。只需确定您希望为每个粒度保留数据的时间长度，并使用标准查询来删除过期数据。在以下示例中，我们决定将原始数据保留一天，每分钟的聚合保留一个月:

.. code-block:: plpgsql

  DELETE FROM http_request WHERE ingest_time < now() - interval '1 day';
  DELETE FROM http_request_1min WHERE ingest_time < now() - interval '1 month';

在生产中，您可以将这些查询包装在一个函数中，并在定时任务中每分钟调用一次。

通过在Citus散列分布之上使用表范围分区，数据到期可以更快。请参阅 :ref:`timeseries` 部分的详细示例。

这些都是基础！我们提供了一个体系结构，用于摄入HTTP事件，然后将这些事件汇总到预先聚合的表单中。这样，您既可以存储原始事件，也可以使用亚秒级查询为分析仪表板提供驱动。

接下来的部分将扩展到基本体系结构，并向您展示如何解决经常出现的问题。

近似的不同计数
---------------------------

HTTP分析中的一个常见问题是处理 :ref:`近似的不同计数 <count_distinct>`：上个月访问过您网站的唯一身份访问者数量是多少？*精确地* 回答这个问题需要在汇总表中存储所有以前看过的访问者的列表，这是一个非常大量的数据。然而，一个近似的答案更易于管理。

称为hyperloglog或HLL的数据类型可以近似回答查询;
它需要非常小的空间来告诉您一个集合中大约有多少个惟一的元素。
它需要一个惊人的小空间来告诉你大约有多少独特元素在一组中。它的准确度可以调整。我们将使用它, 仅使用1280字节的数据，最多可以计算数百亿的唯一身份访问者，最多只有2.2％的误差。

如果要运行全局查询，则会出现等效问题，例如上个月访问过任何客户端站点的唯一IP地址数。如果没有HLL，此查询涉及将工作人员的IP地址列表发送给协调员，以便进行重复数据删除。这既是大量的网络流量，也是大量的计算。通过使用HLL，您可以大大提高查询速度。

首先，您必须安装HLL扩展; `github repo <https://github.com/citusdata/postgresql-hll>`_ 有说明。接下来，您需要启用它：

.. code-block:: sql

  --------------------------------------------------------
  -- Run on all nodes ------------------------------------

  CREATE EXTENSION hll;

.. note::

  这在Citus Cloud上是不必要的，它已经安装了HLL，以及其他有用的 :ref:`cloud_extensions`。

现在，我们已准备好使用HLL跟踪汇总中的IP地址。首先在汇总表中添加一列。

.. code-block:: sql

  ALTER TABLE http_request_1min ADD COLUMN distinct_ip_addresses hll;

接下来使用我们的自定义聚合来填充列。只需将它添加到我们的汇总函数的查询中：

.. code-block:: diff

  @@ -1,10 +1,12 @@
    INSERT INTO http_request_1min (
      site_id, ingest_time, request_count,
      success_count, error_count, average_response_time_msec,
  +   distinct_ip_addresses
    ) SELECT
      site_id,
      minute,
      COUNT(1) as request_count,
      SUM(CASE WHEN (status_code between 200 and 299) THEN 1 ELSE 0 END) as success_count,
      SUM(CASE WHEN (status_code between 200 and 299) THEN 0 ELSE 1 END) as error_count,
      SUM(response_time_msec) / COUNT(1) AS average_response_time_msec,
  +   hll_add_agg(hll_hash_text(ip_address)) AS distinct_ip_addresses
    FROM http_request

仪表板查询稍微复杂一点，您必须通过调用 ``hll_cardinality`` 函数读出不同数量的IP地址：

.. code-block:: sql

  SELECT site_id, ingest_time as minute, request_count,
         success_count, error_count, average_response_time_msec,
         hll_cardinality(distinct_ip_addresses) AS distinct_ip_address_count
    FROM http_request_1min
   WHERE ingest_time > date_trunc('minute', now()) - interval '5 minutes';

HLL不仅速度更快，而且可以让您做以前无法做到的事情。假设我们完成了汇总，但我们没有使用HLL，而是保存了确切的唯一计数。这样可以正常工作，但您无法回答诸如“在过去一周内我们丢弃原始数据的过程中有多少个不同的会话？”。

使用HLL，这很容易。您可以使用以下查询计算一段时间内的不同IP计数：

.. code-block:: sql

  SELECT hll_cardinality(hll_union_agg(distinct_ip_addresses))
  FROM http_request_1min
  WHERE ingest_time > date_trunc('minute', now()) - '5 minutes'::interval;

您可以 `在项目的GitHub存储库 <https://github.com/aggregateknowledge/postgresql-hll>`_ 中找到有关HLL的更多信息。

使用JSONB的非结构化数据
----------------------------

Citus与Postgres对非结构化数据类型的内置支持配合得很好。为了证明这一点，让我们跟踪来自每个国家的访客数量。
使用半结构数据类型可以使您无需为每个国家/地区添加列，并最终获得具有数百个稀疏填充列的行。
我们有一篇 `一篇博客文章 <https://www.citusdata.com/blog/2016/07/14/choosing-nosql-hstore-json-jsonb/>`_，解释了用于半结构化数据的格式。文章推荐JSONB，这里我们将演示如何将JSONB列合并到您的数据模型中。

首先，将新列添加到汇总表：

.. code-block:: sql

  ALTER TABLE http_request_1min ADD COLUMN country_counters JSONB;

接下来，通过修改汇总函数将其包含在汇总中：

.. code-block:: diff

  @@ -1,14 +1,19 @@
    INSERT INTO http_request_1min (
      site_id, ingest_time, request_count,
      success_count, error_count, average_response_time_msec,
  +   country_counters
    ) SELECT
      site_id,
      minute,
      COUNT(1) as request_count,
      SUM(CASE WHEN (status_code between 200 and 299) THEN 1 ELSE 0 END) as success_c
      SUM(CASE WHEN (status_code between 200 and 299) THEN 0 ELSE 1 END) as error_cou
      SUM(response_time_msec) / COUNT(1) AS average_response_time_msec,
  - FROM http_request
  +   jsonb_object_agg(request_country, country_count) AS country_counters
  + FROM (
  +   SELECT *,
  +     count(1) OVER (
  +       PARTITION BY site_id, date_trunc('minute', ingest_time), request_country
  +     ) AS country_count
  +   FROM http_request
  + ) h

现在，如果你想在你的仪表盘中得到来自美国的请求数量，你可以修改仪表盘查询如下:

.. code-block:: sql

  SELECT
    request_count, success_count, error_count, average_response_time_msec,
    COALESCE(country_counters->>'USA', '0')::int AS american_visitors
  FROM http_request_1min
  WHERE ingest_time > date_trunc('minute', now()) - '5 minutes'::interval;
