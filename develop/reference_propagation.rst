.. _manual_prop:

手动查询传播
==================

当用户发出查询时，Citus协调者将其划分为较小的查询片段，其中每个查询片段可以在工作者分片上独立运行。这允许Citus在群集中分发每个查询。

但是，查询被分区为片段的方式（以及传播哪些查询)因查询类型而异。在某些高级情况下，手动控制这种行为非常有用。Citus提供实用函数将SQL传播给工作者、分片或placement。

手动查询传播绕过协调者逻辑，锁定和任何其他一致性检查。这些函数作为最后的手段可用于允许Citus本身不运行的语句。请谨慎使用它们以避免数据不一致和死锁。

.. _worker_propagation:

在所有工作者运行
---------------------


执行的最小粒度级别是广播一个语句，以便在所有工作人员上执行。
这对于查看整个工作者数据库的属性或在整个群集中统一创建UDFs非常有用。例如：

.. code-block:: postgresql

  -- Make a UDF available on all workers
  SELECT run_command_on_workers($cmd$ CREATE FUNCTION ... $cmd$);

  -- List the work_mem setting of each worker database
  SELECT run_command_on_workers($cmd$ SHOW work_mem; $cmd$);

.. note::

  本节中的 :code:`run_command_on_workers` 函数和其他手动传播命令只能运行返回单列和单行的查询。

在所有分片上运行
---------------------

下一个粒度级别是在一个特定分布式表的所有分片上运行命令。它可能很有用, 例如，在直接读取工作者表的属性。在工作者节点上本地运行的查询可以完全访问元数据, 比如表统计等。

该 :code:`run_command_on_shards` 函数将SQL命令应用于每个分片，其中分片名称用于在命令中进行插值。
下面是一个示例，通过使用每个工作者上的pg_class表估计每个分片的行数来估计一个分布式表的行数。
注意,  :code:`%s` 将被替换为每个碎片的名称。

.. code-block:: postgresql

  -- Get the estimated row count for a distributed table by summing the
  -- estimated counts of rows for each shard.
  SELECT sum(result::bigint) AS estimated_count
    FROM run_command_on_shards(
      'my_distributed_table',
      $cmd$
        SELECT reltuples
          FROM pg_class c
          JOIN pg_catalog.pg_namespace n on n.oid=c.relnamespace
         WHERE(n.nspname || '.' || relname)::regclass = '%s'::regclass
           AND n.nspname NOT IN('citus', 'pg_toast', 'pg_catalog')
      $cmd$
    );


在所有位置上运行
---------------------

最精细的执行级别是跨所有分片及其副本（也称为 :ref:`位置 <placements>`)运行命令。它对于运行数据修改命令非常有用，这些命令必须应用于每个副本以确保一致性。

例如，假设分布式表具有 :code:`updated_at` 字段，并且我们想要“触摸”所有行，以便在特定时间将它们标记为已更新。协调者上的普通UPDATE语句需要分发列的过滤器，但我们可以跨所有分片和副本手动传播更新：

.. code-block:: postgresql

  -- note we're using a hard-coded date rather than
  -- a function such as "now()" because the query will
  -- run at slightly different times on each replica

  SELECT run_command_on_placements(
    'my_distributed_table',
    $cmd$
      UPDATE %s SET updated_at = '2017-01-01';
    $cmd$
  );

:code:`run_command_on_placements` 的一个有用伙伴是 :code:`run_command_on_colocated_placements`。它将 :ref:`位于同一位置 <colocation>` 分布式表的 *两个* 位置的名称插入到一个查询中。位置对总是被选择为本地的同一个工作者，在那里的使用可以覆盖完整的SQL。因此，我们可以使用像触发器这样的高级SQL功能来关联表：

.. code-block:: postgresql

  -- Suppose we have two distributed tables
  CREATE TABLE little_vals(key int, val int);
  CREATE TABLE big_vals   (key int, val int);
  SELECT create_distributed_table('little_vals', 'key');
  SELECT create_distributed_table('big_vals',    'key');

  -- We want to synchronise them so that every time little_vals
  -- are created, big_vals appear with double the value
  --
  -- First we make a trigger function on each worker, which will
  -- take the destination table placement as an argument
  SELECT run_command_on_workers($cmd$
    CREATE OR REPLACE FUNCTION embiggen() RETURNS TRIGGER AS $$
      BEGIN
        IF(TG_OP = 'INSERT') THEN
          EXECUTE format(
            'INSERT INTO %s(key, val) SELECT($1).key,($1).val*2;',
            TG_ARGV[0]
          ) USING NEW;
        END IF;
        RETURN NULL;
      END;
    $$ LANGUAGE plpgsql;
  $cmd$);

  -- Next we relate the co-located tables by the trigger function
  -- on each co-located placement
  SELECT run_command_on_colocated_placements(
    'little_vals',
    'big_vals',
    $cmd$
      CREATE TRIGGER after_insert AFTER INSERT ON %s
        FOR EACH ROW EXECUTE PROCEDURE embiggen(%s)
    $cmd$
  );

限制
------

* 多语句事务没有防止死锁的安全措施。
* 没有针对中间查询失败的保护措施以及由此导致的不一致。
* 查询结果缓存在内存中; 这些函数无法处理非常大的结果集。
* 如果函数不能连接到节点，则会提前出错。
* 你可以做很糟糕的事情！
