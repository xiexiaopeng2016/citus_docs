.. _append_distribution:

追加分步
===================

.. note::

  追加分配是一种需要谨慎使用的专业技术。对于大多数情况，散列分布是更好的选择。

虽然Citus最常见的用例涉及散列数据分发，但它也可以按时间顺序在可变数量的分片中分配时间序列数据。本节提供了加载，删除和操作时间序列数据的简短参考。

顾名思义，基于追加的分布更适合append-only用例。这通常包括基于事件的数据，它以按时间排顺的系列到达。然后，您可以按时间分布最大的表，并以N分钟为间隔将事件批量加载到Citus中。该数据模型可以推广到许多时间序列用例; 例如，网站日志文件中的每一行，机器活动日志或聚合的网站事件。基于附加的分布支持更有效的范围查询。这是因为给定分布键上的范围查询，Citus查询计划器可以轻松确定哪些分片与该范围重叠，并仅将查询发送到相关分片。

基于散列的分布更适合于您想要执行实时插入以及对数据进行分析的情况，或者您想要通过非有序的列进行分布的情况(例如, 用户id)。此数据模型与实时分析用例相关; 例如，移动应用程序中的操作，用户网站事件或社交媒体分析。在这种情况下，Citus将为所有创建的分片维护最小和最大散列范围。每当一行被插入，更新或删除时，Citus都会将查询重定向到正确的分片并在本地发出。此数据模型更适合于执行共址连接以及涉及分布列上基于相等过滤器的查询。

Citus使用稍微不同的语法来创建和操作append和hash分布表。此外，表中支持的操作因所选的分布方法而异。在接下来的部分中，我们将介绍创建追加分布式表的语法，并描述可以对它们执行的操作。

创建和分布表
---------------------------------

.. note::

  下面的说明假定PostgreSQL安装已经在您的path中。如果没有，则需要将其添加到PATH环境变量中。例如：

  .. code-block:: bash

      export PATH=/usr/lib/postgresql/9.6/:$PATH


我们使用github事件数据集来说明下面的命令。您可以通过运行以下命令下载该数据集：

.. code-block:: bash

    wget http://examples.citusdata.com/github_archive/github_events-2015-01-01-{0..5}.csv.gz
    gzip -d github_events-2015-01-01-*.gz

要创建追加分布式表，您需要首先定义表架构。为此，您可以使用 `CREATE TABLE <http://www.postgresql.org/docs/current/static/sql-createtable.html>`_ 语句定义表, 与定义常规PostgreSQL表的方式相同。

.. code-block:: postgresql

    -- psql -h localhost -d postgres

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

接下来，您可以使用create_distributed_table()函数将表标记为追加分布式表, 并指定其分布列。

.. code-block:: postgresql

    SELECT create_distributed_table('github_events', 'created_at', 'append');

此函数通知Citus应该在created_at列上来分发github_events表。
这个函数通知Citus, github_events表应该在created_at列上用append方式分布。
注意，此方法不强制执行特定的分布;
它只告诉数据库在每个分片中保留created_at列的最小值和最大值，稍后数据库将使用这些分片来优化查询。
请注意，此方法不强制执行特定分发; 它只是告诉数据库在每个分片中保留created_at列的最小值和最大值，稍后, 数据库用这些值来优化查询。

到期数据
---------------

在追加分布中，用户通常只想跟踪过去几个月/年的数据。
在这种情况下，不再需要的分片仍然占用磁盘空间。
为了解决这个问题，Citus提供了一个用户定义的函数master_apply_delete_command()来删除旧的分片。
该函数将 `DELETE <http://www.postgresql.org/docs/current/static/sql-delete.html>`_ 命令作为输入，并删除与删除条件匹配的所有分片及其元数据。

该函数使用分片元数据来决定是否需要删除分片，因此它要求DELETE语句中的WHERE子句位于分发列上。如果未指定条件，则选择所有分片进行删除。然后，UDF连接到工作节点，并为需要删除的所有分片发出DROP命令。如果特定分片副本的删除查询失败，则该副本将标记为TO DELETE。标记为TO DELETE的分片副本不会被考虑用于将来的查询，可以在以后清除。

下面的示例从github_events表中删除那些包含created_at> ='2015-01-01 00:00:00'的所有行的分片。请注意，该表分布在created_at列上。

.. code-block:: postgresql

    SELECT * from master_apply_delete_command('DELETE FROM github_events WHERE created_at >= ''2015-01-01 00:00:00''');
     master_apply_delete_command
    -----------------------------
                               3
   (1 row)

要了解该函数，其参数及其用法的更多信息，请访问我们文档中的 :ref:`user_defined_functions` 部分。请注意，此功能仅删除分片中的完整分片而不删除单个行。如果您的用例需要实时删除单个行，请参阅以下有关删除数据的部分。

删除数据
---------------

在Citus集群中修改或删除行的最灵活方法是使用常规SQL语句：

.. code-block:: postgresql

  DELETE FROM github_events
  WHERE created_at >= '2015-01-01 00:03:00';

与master_apply_delete_command不同，标准SQL在行, 而不是分片级别工作，以修改或删除与where子句中的条件匹配的所有行。它会删除行，无论它们是否包含整个分片。

删除表
---------------

您可以使用标准 `DROP TABLE <http://www.postgresql.org/docs/current/static/sql-droptable.html>`_ 命令删除追加分布式表。与常规表一样，DROP TABLE删除目标表存在的所有索引，规则，触发器和约束。此外，它还会删除工作节点上的分片并清除其元数据。

.. code-block:: postgresql

    DROP TABLE github_events;

数据加载
------------

Citus支持两种方法将数据加载到追加分布式表中。第一个适用于文件的批量加载，并涉及使用 \\copy 命令。对于需要较小的增量数据加载的用例，Citus提供两个用户定义的函数。我们将在下面描述每种方法及其用法。

使用 \\copy 进行批量加载
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

`\\copy <http://www.postgresql.org/docs/current/static/app-psql.html#APP-PSQL-META-COMMANDS-COPY>`_ 命令用于将数据从一个文件复制到一个分布式表，同时自动处理复制和失败。
您也可以使用服务器端 `COPY命令 <http://www.postgresql.org/docs/current/static/sql-copy.html>`_ 。
在示例中，我们使用psql中的\\copy命令，它发送COPY .. FROM STDIN到服务器, 并读取客户端上的文件，而来自文件的COPY将读取服务器上的文件。

您可以在协调者和任何工作者上使用\\copy。从工作者中使用它时，需要添加master_host选项。在幕后，\\copy首先使用提供的master_host选项打开与协调者的连接，并使用master_create_empty_shard创建新的分片。然后，该命令连接到工作者并将数据复制到副本中，直到大小达到shard_max_size，此时将创建另一个新分片。最后，该命令获取分片的统计信息并更新元数据。

.. code-block:: psql

    SET citus.shard_max_size TO '64MB';
    \copy github_events from 'github_events-2015-01-01-0.csv' WITH(format CSV, master_host 'coordinator-host')

Citus为每个新分片分配一个唯一的分片ID，并且其所有副本都具有相同的分片ID。每个分片在工作节点上表示为名为'tablename_shardid'的常规PostgreSQL表，其中tablename是分布式表的名称，shardid是分配给该分片的唯一ID。可以连接到工作者postgres实例以查看或运行各个分片上的命令。

默认情况下，\\copy命令的行为依赖两个配置参数。这些被称为citus.shard_max_size和citus.shard_replication_factor。

(1) **citus.shard_max_size :-** 此参数确定使用\\copy创建的分片的最大大小，默认为1GB。如果文件大于此参数，\\copy会将其分解为多个分片。
(2) **citus.shard_replication_factor :-** 此参数确定每个分片复制到的节点数，默认为1。如果希望Citus自动复制数据并提供容错功能，请将其设置为2。如果运行大型集群并更频繁地观察节点故障，您可能希望将该因子提高得更高。

.. note::

    配置设置citus.shard_replication_factor只能在协调器节点上设置。

请注意，您可以通过单独的数据库连接或从不同的节点并行加载多个文件。值得注意的是，\\copy始终创建至少一个分片，并且不会附加到现有分片。您可以使用下面描述的方法附加到以前创建的分片。您可以使用下面描述的方法追加到之前创建的分片后面。

.. note::

    跨分片没有快照隔离的概念，这意味着与COPY同时运行的多分片SELECT可能会在某些分片上看到它的提交，但在其他分片上却没有。如果用户正在存储事件数据，他可能偶尔会观察到最近数据中的小间隙。如果这是一个问题，则由应用程序来处理(例如，从查询中排除最近的数据，或使用一些锁)。

    如果COPY无法为分片位置打开连接，则其行为方式与INSERT相同，即将位置标记为非活动状态，除非没有更多活动位置。如果在连接后发生任何其他故障，则回滚事务，因此不会进行元数据更改。

通过附加到现有分片来增量加载
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

\\copy命令在使用时始终会创建一个新的分片，最适合批量加载数据。使用\\copy加载较小的数据增量将导致许多小分片，这可能不是理想的。为了允许较小的增量加载到附加分布式表中，Citus提供了2个用户定义的函数。它们是master_create_empty_shard()和master_append_table_to_shard()。

master_create_empty_shard()可用于为表创建新的空分片。此函数还将空分片复制到citus.shard_replication_factor个节点，类似\\copy命令。

master_append_table_to_shard()可用于将PostgreSQL表的内容附加到现有分片。这允许用户控制将哪些行到切分。它还返回分片填充率，这有助于确定是否应将更多数据附加到此分片或是否应创建新分片

要使用上述功能，您可以先将传入数据插入常规PostgreSQL表中。然后，您可以使用 master_create_empty_shard()创建空分片。然后，使用 master_append_table_to_shard()，你可以将staging表的内容附加到指定的分片，随后从staging表中删除数据。一旦append函数返回的分片填充率接近1，您就可以创建一个新分片并开始追加到新分片。

.. code-block:: postgresql

    SELECT * from master_create_empty_shard('github_events');
    master_create_empty_shard
    ---------------------------
                    102089
   (1 row)

    SELECT * from master_append_table_to_shard(102089, 'github_events_temp', 'master-101', 5432);
    master_append_table_to_shard
    ------------------------------
            0.100548
   (1 row)

要了解有关这两个UDF及其参数和用法的更多信息，请访问文档的 :ref:`user_defined_functions` 部分。

提高数据加载性能
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

上面描述的方法使您能够实现高的批量负载率，这对于大多数用例来说已经足够了。如果需要更高的数据加载速率，可以通过多种方式使用上述功能并编写脚本以更好地控制分片和数据加载。下一节将介绍如何更快地完成任务。

缩放数据摄取
----------------------

如果您的用例不需要实时摄取，那么使用追加分布式表将为您提供最高的摄取率。这种方法更适用于使用时间序列数据的用例，数据库可能落后几分钟或更长时间。

协调者节点批量摄取(100k/s-200k/s)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

要将数据摄取到追加分布式表中，可以使用 `COPY <http://www.postgresql.org/docs/current/static/sql-copy.html>`_ 命令，该命令将从您提取的数据中创建新的分片。COPY可以将大于配置的citus.shard_max_size的文件分解为多个分片。附加分布式表的COPY只打开新分片的连接，这意味着它的行为与散列分布式表的COPY略有不同，后者可能打开所有碎片的连接。附加分布式表的COPY命令不会在许多连接上并行地摄取行，但是并行地运行许多命令是安全的。

.. code-block:: psql

    -- Set up the events table
    CREATE TABLE events(time timestamp, data jsonb);
    SELECT create_distributed_table('events', 'time', 'append');

    -- Add data into a new staging table
    \COPY events FROM 'path-to-csv-file' WITH CSV

COPY每次使用时都会创建新的分片，这样可以同时摄取多个文件，但如果查询最终涉及数千个分片，则可能会导致问题。摄取数据的另一种方法是使用 master_append_table_to_shard 函数将其附加到现有分片。要使用 master_append_table_to_shard，需要将数据加载到临时表中，并且需要一些自定义逻辑来选择适当的分片。

.. code-block:: psql

    -- Prepare a staging table
    CREATE TABLE stage_1(LIKE events);
    \COPY stage_1 FROM 'path-to-csv-file WITH CSV

    -- In a separate transaction, append the staging table
    SELECT master_append_table_to_shard(select_events_shard(), 'stage_1', 'coordinator-host', 5432);

下面给出了分片选择函数的示例。它附加到一个分片，直到它的大小大于1GB，然后创建一个新的分片，这有一个缺点，一次只允许一个附加，但优势是限制分片大小。

.. code-block:: postgresql

    CREATE OR REPLACE FUNCTION select_events_shard() RETURNS bigint AS $$
    DECLARE
      shard_id bigint;
    BEGIN
      SELECT shardid INTO shard_id
      FROM pg_dist_shard JOIN pg_dist_placement USING(shardid)
      WHERE logicalrelid = 'events'::regclass AND shardlength < 1024*1024*1024;

      IF shard_id IS NULL THEN
        /* no shard smaller than 1GB, create a new one */
        SELECT master_create_empty_shard('events') INTO shard_id;
      END IF;

      RETURN shard_id;
    END;
    $$ LANGUAGE plpgsql;

创建一个序列来为staging表生成一个惟一的名称也可能很有用。这样，每次摄入都可以独立处理。

.. code-block:: postgresql

    -- Create stage table name sequence
    CREATE SEQUENCE stage_id_sequence;

    -- Generate a stage table name
    SELECT 'stage_'||nextval('stage_id_sequence');

要了解有关 master_append_table_to_shard 和 master_create_empty_shard UDF的更多信息，请访问文档的 :ref:`user_defined_functions` 部分。

工作节点批量摄取(100k/s-1M/s)
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

对于非常高的数据摄取率，数据可以通过工作者进行分段。这种方法横向扩展，提供了最高的摄取率，但使用起来可能更复杂。因此，我们建议仅当您的数据摄取率无法通过前面描述的方法解决时，才尝试这种方法。

附加分布式表通过工作者支持COPY，通过在master_host选项中指定协调者的地址，以及可选的master_port选项（默认为5432)。通过工作者的COPY与通过协调者的COPY具有相同的常规属性，除了初始解析在协调者上没有瓶颈。

.. code-block:: psql

    psql -h worker-host-n -c "\COPY events FROM 'data.csv' WITH(FORMAT CSV, MASTER_HOST 'coordinator-host')"


使用COPY的另一种选择是创建一个staging表, 并使用标准的SQL客户端将其附加到分布式表中,这类似于通过协调器进行数据分段。
使用psql通过工作者分段文件的示例如下：

.. code-block:: bash

    stage_table=$(psql -tA -h worker-host-n -c "SELECT 'stage_'||nextval('stage_id_sequence')")
    psql -h worker-host-n -c "CREATE TABLE $stage_table(time timestamp, data jsonb)"
    psql -h worker-host-n -c "\COPY $stage_table FROM 'data.csv' WITH CSV"
    psql -h coordinator-host -c "SELECT master_append_table_to_shard(choose_underutilized_shard(), '$stage_table', 'worker-host-n', 5432)"
    psql -h worker-host-n -c "DROP TABLE $stage_table"

上面的示例使用 choose_underutilized_shard 函数来选择要追加的分片。为确保并行数据摄取，此功能应在许多不同的分片之间取得平衡。

下面的示例 choose_underutilized_shard 函数随机选择20个最小的分片中的一个，或者如果1GB以下少于20，则创建一个新分片。这允许20个并发附加，允许数据摄取高达100万行/秒(取决于索引，大小，容量)。

.. code-block:: postgresql

    /* Choose a shard to which to append */
    CREATE OR REPLACE FUNCTION choose_underutilized_shard()
    RETURNS bigint LANGUAGE plpgsql
    AS $function$
    DECLARE
      shard_id bigint;
      num_small_shards int;
    BEGIN
      SELECT shardid, count(*) OVER() INTO shard_id, num_small_shards
      FROM pg_dist_shard JOIN pg_dist_placement USING(shardid)
      WHERE logicalrelid = 'events'::regclass AND shardlength < 1024*1024*1024
      GROUP BY shardid ORDER BY RANDOM() ASC;

      IF num_small_shards IS NULL OR num_small_shards < 20 THEN
        SELECT master_create_empty_shard('events') INTO shard_id;
      END IF;

      RETURN shard_id;
    END;
    $function$;

同时摄入多个分片的缺点是分片可能跨越更长的时间范围，这意味着特定时间段的查询可能涉及包含该时段之外的大量数据的分片。

除了复制到临时staging表之外，还可以设置工作者上的表, 可以为这些表连续执行INSERT。在这种情况下，必须定期将数据移动到staging表中，然后追加，但这需要更高级的脚本

Citus中的预处理数据
$$$$$$$$$$$$$$$$$$$$$$$$$$$$

传递原始数据的格式通常不同于数据库中使用的模式。
例如, 原始数据可能以日志文件的形式存在, 每一行都是一个JSON对象, 而在数据库表中, 在分开的列中存储公共值更有效率。
此外，分布式表应始终具有分发列。
幸运的是，PostgreSQL是一个非常强大的数据处理工具。在将结果放入staging表之前，可以使用SQL应用任意预处理。

例如，假设我们有以下表模式，并希望从 `githubarchive.org <http://www.githubarchive.org>`_ 加载压缩的JSON日志：

.. code-block:: postgresql

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
    SELECT create_distributed_table('github_events', 'created_at', 'append');


为了加载数据,我们可以下载数据、分解数据、过滤不受支持的行,并使用3个命令提取感兴趣的字段到一个staging表:

.. code-block:: postgresql

    CREATE TEMPORARY TABLE prepare_1(data jsonb);

    -- Load a file directly from Github archive and filter out rows with unescaped 0-bytes
    COPY prepare_1 FROM PROGRAM
    'curl -s http://data.githubarchive.org/2016-01-01-15.json.gz | zcat | grep -v "\\u0000"'
    CSV QUOTE e'\x01' DELIMITER e'\x02';

    -- Prepare a staging table
    CREATE TABLE stage_1 AS
    SELECT(data->>'id')::bigint event_id,
          (data->>'type') event_type,
          (data->>'public')::boolean event_public,
          (data->'repo'->>'id')::bigint repo_id,
          (data->'payload') payload,
          (data->'actor') actor,
          (data->'org') org,
          (data->>'created_at')::timestamp created_at FROM prepare_1;

然后,您可以使用 master_append_append_table_to_shard 函数将该staging表附加到分布式表中。

这种方法在通过工作者进行数据分段时特别有效, 因为预处理本身可以通过在不同的输入数据块上并行运行许多工作程序来扩展。

有关更完整的示例，请参阅 `Interactive Analytics on GitHub Data using PostgreSQL with Citus <https://www.citusdata.com/blog/14-marco/402-interactive-analytics-github-data-using-postgresql-citus>`_.
