---
title: CREATE TASK
sidebar_position: 1
---

import FunctionDescription from '@site/src/components/FunctionDescription';

<FunctionDescription description="引入或更新：v1.2.371"/>

CREATE TASK 语句用于定义一个新任务，该任务将按照预定的时间表或基于任务图（DAG）执行指定的 SQL 语句。

**注意：** 此功能在 Databend Cloud 中开箱即用。

## 语法

```sql
CREATE TASK [ IF NOT EXISTS ] <name>
 [ WAREHOUSE = <string ]
 [ SCHEDULE = { <num> MINUTE | <num> SECOND | USING CRON <expr> <time_zone> } ]
 [ AFTER <string> [ , <string> , ... ]]
 [ WHEN <boolean_expr> ]
 [ SUSPEND_TASK_AFTER_NUM_FAILURES = <num> ]
 [ ERROR_INTEGRATION = <string> ]
 [ COMMENT = '<string_literal>' ]
 [ <session_parameter> = <value> [ , <session_parameter> = <value> ... ] ]
AS
<sql>
```

| 参数                                             | 描述                                                                                                   |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| IF NOT EXISTS                                    | 可选。如果指定，则只有在同名任务不存在时才会创建任务。                                                 |
| name                                             | 任务的名称。这是一个必填字段。                                                                         |
| WAREHOUSE                                        | 可选。指定用于任务的虚拟计算集群。                                                                     |
| SCHEDULE                                         | 可选。定义任务运行的计划。可以按分钟指定，或使用 CRON 表达式及时间区域。                               |
| SUSPEND_TASK_AFTER_NUM_FAILURES                  | 可选。任务连续失败后将自动挂起的次数。                                                                 |
| AFTER                                            | 列出在此任务开始之前必须完成的任务。                                                                   |
| WHEN boolean_expr                                | 任务运行的条件，必须为真。                                                                             |
| [ERROR_INTEGRATION](../16-notification/index.md) | 可选。用于任务错误通知的通知集成名称，应用特定的[任务错误负载](./10-task-error-integration-payload.md) |
| COMMENT                                          | 可选。作为任务注释或描述的字符串字面量。                                                               |
| session_parameter                                | 可选。指定任务运行期间使用的会话参数。                                                                 |
| sql                                              | 任务将执行的 SQL 语句，可以是单个语句或脚本。这是一个必填字段。                                        |

### 使用说明：

- 必须为独立任务或 DAG 中的根任务定义计划；否则，任务仅在手动执行 EXECUTE TASK 时运行。
- 不能为 DAG 中的子任务指定计划。
- 创建任务后，必须执行 ALTER TASK … RESUME 才能根据任务定义中的参数运行任务。
- 当条件仅支持部分 `<boolean_expression>`
  以下在任务 WHEN 子句中支持：

  - [STREAM_STATUS](../../../00-sql-reference/20-system-tables/system-stream-status.md) 支持在 SQL 表达式中进行评估。此函数指示指定流是否包含变更跟踪数据。您可以使用此函数评估指定流是否包含变更数据，然后再开始当前运行。如果结果为 FALSE，则任务不运行。
  - 布尔运算符，如 AND、OR、NOT 等。
  - 数值、字符串和布尔类型之间的类型转换。
  - 比较运算符，如等于、不等于、大于、小于等。

- 从单个表流消费变更数据的多个任务将获取不同的增量。当任务使用 DML 语句消费流中的变更数据时，流会推进偏移量。变更数据不再可供下一个任务消费。目前，我们建议只有一个任务从流中消费变更数据。可以为同一表创建多个流，并由不同的任务消费。
- 任务不会在每次执行时重试；每次执行都是串行的。每个脚本 SQL 逐一执行，没有并行执行。这确保了任务执行的顺序和依赖关系得到维护。
- 基于间隔的任务紧跟固定间隔点。这意味着如果当前任务执行时间超过间隔单位，下一个任务将立即执行。否则，下一个任务将等待下一个间隔单位触发。例如，如果任务定义为 1 秒间隔，一次任务执行需要 1.5 秒，下一个任务将立即执行。如果一次任务执行需要 0.5 秒，下一个任务将等待下一个 1 秒间隔开始。

## 使用示例

```sql
CREATE TASK my_daily_task
 WAREHOUSE = 'compute_wh'
 SCHEDULE = USING CRON '0 9 * * * *' 'America/Los_Angeles'
 COMMENT = '每日汇总任务'
 AS
 INSERT INTO summary_table SELECT * FROM source_table;
```

在此示例中，创建了一个名为 my_daily_task 的任务。它使用 compute_wh 仓库运行一个 SQL 语句，将数据从 source_table 插入到 summary_table 中。任务计划每天上午 9 点太平洋时间运行。

```sql
CREATE TASK IF NOT EXISTS mytask
 WAREHOUSE = 'system'
 SCHEDULE = 2 MINUTE
 SUSPEND_TASK_AFTER_NUM_FAILURES = 3
AS
INSERT INTO compaction_test.test VALUES((1));
```

此示例创建一个名为 mytask 的任务（如果不存在）。任务分配给 system 仓库，并计划每 2 分钟运行一次。如果连续失败三次，任务将自动挂起。任务执行插入操作到 compaction_test.test 表中。

```sql
CREATE TASK IF NOT EXISTS daily_sales_summary
 WAREHOUSE = 'analytics'
 SCHEDULE = 30 SECOND
FROM sales_data
GROUP BY sales_date;
```

在此示例中，创建了一个名为 daily_sales_summary 的任务，具有秒级调度。它计划每 30 秒运行一次。任务使用 'analytics' 仓库，并通过聚合 sales_data 表中的数据计算每日销售汇总。

```sql
CREATE TASK IF NOT EXISTS process_orders
 WAREHOUSE = 'etl'
 AFTER task1, task2
ASINSERT INTO data_warehouse.orders
SELECT * FROM staging.orders;
```

在此示例中，创建了一个名为 process_orders 的任务，并定义为在 task1 和 task2 成功完成后运行。这对于在任务的有向无环图（DAG）中创建依赖关系很有用。任务使用 'etl' 仓库，并将数据从暂存区转移到数据仓库。

```sql
CREATE TASK IF NOT EXISTS hourly_data_cleanup
 WAREHOUSE = 'maintenance'
 SCHEDULE = '0 * * * *'
 WHEN STREAM_STATUS('change_stream') = TRUE
AS
DELETE FROM archived_data
WHERE archived_date < DATEADD(HOUR, -24, CURRENT_TIMESTAMP());

```

在此示例中，创建了一个名为 hourly_data_cleanup 的任务。它使用 maintenance 仓库，并计划每小时运行一次。任务删除 archived_data 表中超过 24 小时的数据。任务仅在 change_stream 流包含变更数据时运行。

```sql
CREATE TASK IF NOT EXISTS mytask
 WAREHOUSE = 'mywh'
 SCHEDULE = 30 SECOND
 ERROR_INTEGRATION = 'myerror'
AS
BEGIN
    BEGIN;
    INSERT INTO mytable(ts) VALUES(CURRENT_TIMESTAMP);
    DELETE FROM mytable WHERE ts < DATEADD(MINUTE, -5, CURRENT_TIMESTAMP());
    COMMIT;
END;
```

在此示例中，创建了一个名为 mytask 的任务。它使用 mywh 仓库，并计划每 30 秒运行一次。任务执行一个包含 INSERT 语句和 DELETE 语句的 BEGIN 块。任务在两个语句执行后提交事务。当任务失败时，将触发名为 myerror 的错误集成。
