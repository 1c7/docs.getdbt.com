---
title: "Apache Spark configurations"
id: "spark-configs"
---

:::caution Heads up!
These docs are a work in progress.

:::

<!----
To-do:
- use the reference doc structure for this article/split into separate articles
--->

## Configuring tables

When materializing a model as `table`, you may include several optional configs:

| Option  | Description                                        | Required?               | Example                  |
|---------|----------------------------------------------------|-------------------------|--------------------------|
| file_format | The file format to use when creating tables (`parquet`, `delta`, `csv`, `json`, `text`, `jdbc`, `orc`, `hive` or `libsvm`). | Optional | `parquet`|
| location_root  | The created table uses the specified directory to store its data. The table alias is appended to it. | Optional                | `/mnt/root`              |
| partition_by  | Partition the created table by the specified columns. A directory is created for each partition. | Optional                | `date_day`              |
| clustered_by  | Each partition in the created table will be split into a fixed number of buckets by the specified columns. | Optional               | `country_code`              |
| buckets  | The number of buckets to create while clustering | Required if `clustered_by` is specified                | `8`              |

## Incremental models

The [`incremental_strategy` config](configuring-incremental-models#what-is-an-incremental_strategy) controls how dbt builds incremental models, and it can be set to one of two values:
 - `insert_overwrite` (default)
 - `merge` (Delta Lake only)

### The `insert_overwrite` strategy

Apache Spark does not natively support `delete`, `update`, or `merge` statements. As such, Spark's default incremental behavior is different [from the standard](configuring-incremental-models).

To use incremental models, specify a `partition_by` clause in your model config. dbt will run an [atomic `insert overwrite` statement](https://spark.apache.org/docs/3.0.0-preview/sql-ref-syntax-dml-insert-overwrite-table.html) that dynamically replaces all partitions included in your query. Be sure to re-select _all_ of the relevant data for a partition when using this incremental strategy.

<Tabs
  defaultValue="source"
  values={[
    { label: 'Source code', value: 'source', },
    { label: 'Run code', value: 'run', },
  ]
}>
<TabItem value="source">

<File name='spark_incremental.sql'>

```sql
{{ config(
    materialized='incremental',
    partition_by=['date_day'],
    file_format='parquet'
) }}

/*
  Every partition returned by this query will be overwritten
  when this model runs
*/

with new_events as (

    select * from {{ ref('events') }}

    {% if is_incremental() %}
    where date_day >= date_add(current_date, -1)
    {% endif %}

)

select
    date_day,
    count(*) as users

from events
group by 1
```

</File>
</TabItem>
<TabItem value="run">

<File name='spark_incremental.sql'>

```sql
create temporary view spark_incremental__dbt_tmp as

    with new_events as (

        select * from analytics.events


        where date_day >= date_add(current_date, -1)


    )

    select
        date_day,
        count(*) as users

    from events
    group by 1

;

insert overwrite table analytics.spark_incremental
    partition (date_day)
    select `date_day`, `users` from spark_incremental__dbt_tmp
```

</File>
</TabItem>
</Tabs>

### The `merge` strategy

:::info New in dbt-spark v0.15.3

This functionality is new in dbt-spark v0.15.3. See [installation instructions](spark-profile#installation-and-distribution)

:::

There are three prerequisites for the `merge` incremental strategy:
- Creating the table in Delta file format
- Using Databricks Runtime 5.1 and above
- Specifying a `unique_key`

dbt will run an [atomic `merge` statement](https://docs.databricks.com/spark/latest/spark-sql/language-manual/merge-into.html) which looks nearly identical to the default merge behavior on Snowflake and BigQuery.

<Tabs
  defaultValue="source"
  values={[
    { label: 'Source code', value: 'source', },
    { label: 'Run code', value: 'run', },
  ]
}>
<TabItem value="source">

<File name='delta_incremental.sql'>

```sql
{{ config(
    materialized='incremental',
    file_format='delta',
    unique_key='user_id',
    incremental_strategy='merge'
) }}

with new_events as (

    select * from {{ ref('events') }}

    {% if is_incremental() %}
    where date_day >= date_add(current_date, -1)
    {% endif %}

)

select
    user_id,
    max(date_day) as last_seen

from events
group by 1
```

</File>
</TabItem>
<TabItem value="run">

<File name='delta_incremental.sql'>

```sql
create temporary view delta_incremental__dbt_tmp as

    with new_events as (

        select * from analytics.events


        where date_day >= date_add(current_date, -1)


    )

    select
        user_id,
        max(date_day) as last_seen

    from events
    group by 1

;

merge into analytics.delta_incremental as DBT_INTERNAL_DEST
    using delta_incremental__dbt_tmp as DBT_INTERNAL_SOURCE
    on DBT_INTERNAL_SOURCE.user_id = DBT_INTERNAL_DEST.user_id
    when matched then update set *
    when not matched then insert *
```

</File>

</TabItem>
</Tabs>

## Persisting model descriptions

Relation-level docs persistence is supported in dbt v0.17.0. For more
information on configuring docs persistence, see [the docs](resource-configs/persist_docs).

When the `persist_docs` option is configured appropriately, you'll be able to
see model descriptions in the `Comment` field of `describe [table] extended`
or `show table extended in [database] like '*'`.

## Always `schema`, never `database`

:::info New in dbt-spark v0.17.0

This is a breaking change in dbt-spark v0.17.0. See [installation instructions](spark-profile#installation-and-distribution)

:::

Apache Spark uses the terms "schema" and "database" interchangeably. dbt understands
`database` to exist at a higher level than `schema`. As such, you should _never_
use or set `database` as a node config or in the target profile when running dbt-spark.

If you want to control the schema/database in which dbt will materialize models,
use the `schema` config and `generate_schema_name` macro _only_.

## Databricks configurations

To access features exclusive to Databricks runtimes, such as 
[snapshots](snapshots) and the `merge` incremental strategy, you will want to
use the Delta file format when materializing models as tables.

It's quite convenient to do this by setting a top-level configuration in your
project file:

<File name='dbt_project.yml'>

```yml
models:
  +file_format: delta
  
seeds:
  +file_format: delta
  
snapshots:
  +file_format: delta
```

</File>