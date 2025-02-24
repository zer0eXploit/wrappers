[ClickHouse](https://clickhouse.com/) is a fast open-source column-oriented database management system that allows generating analytical data reports in real-time using SQL queries.

The ClickHouse Wrapper allows you to read and write data from ClickHouse within your Postgres database.

## Supported Data Types

| Postgres Type      | ClickHouse Type   |
| ------------------ | ----------------- |
| boolean            | UInt8             |
| smallint           | Int16             |
| integer            | UInt16            |
| integer            | Int32             |
| bigint             | UInt32            |
| bigint             | Int64             |
| bigint             | UInt64            |
| real               | Float32           |
| double precision   | Float64           |
| text               | String            |
| date               | Date              |
| timestamp          | DateTime          |

## Preparation

Before you get started, make sure the `wrappers` extension is installed on your database:

```sql
create extension if not exists wrappers;
```

and then create the foreign data wrapper:

```sql
create foreign data wrapper clickhouse_wrapper
  handler click_house_fdw_handler
  validator click_house_fdw_validator;
```

### Secure your credentials (optional)

By default, Postgres stores FDW credentials inide `pg_catalog.pg_foreign_server` in plain text. Anyone with access to this table will be able to view these credentials. Wrappers is designed to work with [Vault](https://supabase.com/docs/guides/database/vault), which provides an additional level of security for storing credentials. We recommend using Vault to store your credentials.

```sql
-- Create a secure key using pgsodium:
select pgsodium.create_key(name := 'clickhouse');

-- Save your ClickHouse credential in Vault and retrieve the `key_id`
insert into vault.secrets (secret, key_id) values (
  'tcp://default:@localhost:9000/default',
  (select id from pgsodium.valid_key where name = 'clickhouse')
) returning key_id;
```

### Connecting to ClickHouse

We need to provide Postgres with the credentials to connect to ClickHouse, and any additional options. We can do this using the `create server` command:

=== "With Vault"

    ```sql
    do $$
    declare
      key_id text;
    begin
      select id into key_id from pgsodium.valid_key where name = 'clickhouse' limit 1;

      execute format(
        E'create server clickhouse_server \n'
        '   foreign data wrapper clickhouse_server \n'
        '   options (conn_string_id ''%s'');',
        key_id
      );
    end $$;
    ```

=== "Without Vault"

    ```sql
    create server clickhouse_server
      foreign data wrapper clickhouse_wrapper
      options (
        conn_string 'tcp://default:@localhost:9000/default'
      );
    ```

## Creating Foreign Tables

The ClickHouse Wrapper supports data reads and writes from ClickHouse.

| Integration | Select            | Insert            | Update            | Delete            | Truncate          |
| ----------- | :----:            | :----:            | :----:            | :----:            | :----:            |
| ClickHouse  | :white_check_mark:| :white_check_mark:| :white_check_mark:| :white_check_mark:| :x:               |

For example:

```sql
create foreign table my_clickhouse_table (
  id bigint,
  name text
)
  server clickhouse_server
  options (
    table 'people'
  );
```

### Foreign table options

The full list of foreign table options are below:

- `table` - Source table name in ClickHouse, required.

   This can also be a subquery enclosed in parentheses, for example,

   ```sql
   table '(select * from my_table)'
   ```

   [Parametrized view](https://clickhouse.com/docs/en/sql-reference/statements/create/view#parameterized-view) is also supported in the subquery. In this case, you need to define a column for each parameter and use `where` to pass values to them. For example,

   ```sql
    create foreign table test_vw (
      id bigint,
      col1 text,
      col2 bigint,
      _param1 text,
      _param2 bigint
    )
      server clickhouse_server
      options (
        table '(select * from my_view(column1=${_param1}, column2=${_param2}))'
      );

    select * from test_vw where _param1='aaa' and _param2=32;
   ```

- `rowid_column` - Primary key column name, optional for data scan, required for data modify

## Examples

Some examples on how to use ClickHouse foreign tables.

### Basic example

This will create a "foreign table" inside your Postgres database called `people`: 

```sql
-- Run below SQLs on ClickHouse to create source table
drop table if exists people;
create table people (
    id Int64, 
    name String
)
engine=MergeTree()
order by id;

-- Add some test data
insert into people values (1, 'Luke Skywalker'), (2, 'Leia Organa'), (3, 'Han Solo');
```

Create foreign table on Postgres database:

```sql
create foreign table people (
  id bigint,
  name text
)
  server clickhouse_server
  options (
    table 'people'
  );

-- data scan
select * from people;

-- data modify
insert into people values (4, 'Yoda');
update people set name = 'Princess Leia' where id = 2;
delete from people where id = 3;
```
