---
author: "Rob Ashton"
date: 2019-08-09
title: Querying for foreign key constraints
best: false
---

Over the past few months we've been developing a [tool to assist with database import from R](https://github.com/vimc/dettl). As part of that we want to automate some of the import steps to both reduce the development work needed for each import and to make code review easier. 

We want to be able to take a list of data frames in R and append those to the relevant tables in the database automatically. Some of the tables contain columns with foreign key constraints so need to ensure that any links between data frames in R are reflected in the database once the data has been uploaded. The difficulty arises when we are referencing autogenerated columns and we do not know the value the autogenerated column will take once the data is uploaded. We've decided to use a temporary identifier for the columns within R which we will update after the referenced table is updated. To enable this we need to know which columns of which tables are referenced by a foreign key constraint. This blog post covers how to query PostgreSQL and SQLite databases to retrieve that information.

## PostgreSQL

Postgres databases capture schema metadata within [catalogs](https://www.postgresql.org/docs/9.2/catalogs-overview.html). These catalogs are just regular tables which we can query as normal. We're going to use three different tables to extract constraint metadata in a readable form:

* [`pg_constraint`](https://www.postgresql.org/docs/9.2/catalog-pg-constraint.html) - contains data about constraints of different types (check, primary key, unique, foreign key and exclusion) for all tables
* [`pg_class`](https://www.postgresql.org/docs/9.2/catalog-pg-class.html) - contains data about tables in the database (it also has information about other relations e.g. sequences, views etc. but we're only interested in the tables here)
* [`pg_attribute`](https://www.postgresql.org/docs/9.2/catalog-pg-attribute.html) - contains information about table columns with one entry for every column of every table in the database

From these the fields we are most interested in are:

### `pg_constraint`
* `oid` - row identifier
* `conname` - constraint name
* `contype` - constraint type, `f` = foreign key constraint, `p` = primary key constraint, etc.
* `conkey` - List of constrained columns
* `conrelid` - `oid` of the table this constraint is on
* `confkey` - List of referenced columns for foreign key
* `confrelid` - `oid` of the referenced table for a foreign key

### `pg_class`
* `oid` - row identifier
* `relname` - the name of the table

### `pg_attribute`
* `attname` - the column name
* `attrelid` - the oid the table belongs to
* `attnum` - the number of the column

Let's set up an example DB with some foreign key constraints. We create 3 tables with 3 foreign key constraints:

```
CREATE TABLE region (
  name TEXT PRIMARY KEY,
  parent TEXT,
  FOREIGN KEY (parent) REFERENCES region(name)
);
CREATE TABLE street (
  name TEXT PRIMARY KEY
);
CREATE TABLE address (
  street TEXT,
  region TEXT,
  FOREIGN KEY (street) REFERENCES street(name),
  FOREIGN KEY (region) REFERENCES region(name)
);
```

<img src="/img/example_db.png" alt="png of database schema"/>

We want to develop a query to pull out the information in a readable form from the above tables. This involves getting the list of constraints from `pg_constraint` then joining `pg_class` and `pg_attribute` on `oid` to pull out the name of the table and column. The difficulty is that conkey and confkey fields are both lists so we need to use `UNNEST` to expand these to get a single row for each constraint. We initially tried to use a `LATERAL` join to retrieve this information from the database using this [stack overflow answer](https://dba.stackexchange.com/a/218969). We're using [Travis](https://travis-ci.org) as our CI system which makes PostgreSQL 9.2 available by default. `LATERAL` join was only introduced in version 9.4 so we need to adapt the query to work with older versions of postgres.

We can use common table expressions (`WITH` clauses) to do this. We need two expressions to get the `conkey` and `confkey` with the corresponding `oid`:
```
SELECT oid, unnest(confkey) as confkey
FROM pg_constraint;
```

```
SELECT oid, unnest(conkey) as conkey
FROM pg_constraint;
```

Then we `LEFT JOIN` to these expressions in the main query to get a row for every combination of constrained column to referenced column to get the full query:

```
WITH unnested_confkey AS (
  SELECT oid, unnest(confkey) as confkey
  FROM pg_constraint
),
unnested_conkey AS (
  SELECT oid, unnest(conkey) as conkey
  FROM pg_constraint
)
select
  c.conname                   AS constraint_name,
  c.contype                   AS constraint_type,
  tbl.relname                 AS constraint_table,
  col.attname                 AS constraint_column,
  referenced_tbl.relname      AS referenced_table,
  referenced_field.attname    AS referenced_column,
  pg_get_constraintdef(c.oid) AS definition
FROM pg_constraint c
LEFT JOIN unnested_conkey con ON c.oid = con.oid
LEFT JOIN pg_class tbl ON tbl.oid = c.conrelid
LEFT JOIN pg_attribute col ON (col.attrelid = tbl.oid AND col.attnum = con.conkey)
LEFT JOIN pg_class referenced_tbl ON c.confrelid = referenced_tbl.oid
LEFT JOIN unnested_confkey conf ON c.oid = conf.oid
LEFT JOIN pg_attribute referenced_field ON (referenced_field.attrelid = c.confrelid AND referenced_field.attnum = conf.confkey)
WHERE c.contype = 'f';
```

Which returns the desired result:

```
   constraint_name   | constraint_type | constraint_table | constraint_column | referenced_table | referenced_column |                  definition                  
---------------------+-----------------+------------------+-------------------+------------------+-------------------+----------------------------------------------
 region_parent_fkey  | f               | region           | parent            | region           | name              | FOREIGN KEY (parent) REFERENCES region(name)
 address_street_fkey | f               | address          | street            | street           | name              | FOREIGN KEY (street) REFERENCES street(name)
 address_region_fkey | f               | address          | region            | region           | name              | FOREIGN KEY (region) REFERENCES region(name)


```

We're restricting the query to only return foreign key constraints but removing the `WHERE` clause will show all constraint types. Care should be taken in interpreting these when constraints are on multiple columns. For example a unique constraint on two separate columns and a multi-column constraint will return similar results; the difference would only be seen in the `constraint_name` and `definition`.  We can demonstrate this by creating two new tables:

```
CREATE TABLE table_with_one_unique (
  first_name TEXT,
  surname TEXT,
  UNIQUE(first_name, surname)
);

CREATE TABLE table_with_two_unique (
  first_name TEXT UNIQUE,
  surname TEXT UNIQUE
);
```
Running our query again, this time selecting only unique constraints using `WHERE c.contype = 'u'` returns:

```
               constraint_name                | constraint_type |   constraint_table    | constraint_column | referenced_table | referenced_column |          definition          
----------------------------------------------+-----------------+-----------------------+-------------------+------------------+-------------------+------------------------------
 table_with_one_unique_first_name_surname_key | u               | table_with_one_unique | surname           |                  |                   | UNIQUE (first_name, surname)
 table_with_one_unique_first_name_surname_key | u               | table_with_one_unique | first_name        |                  |                   | UNIQUE (first_name, surname)
 table_with_two_unique_surname_key            | u               | table_with_two_unique | surname           |                  |                   | UNIQUE (surname)
 table_with_two_unique_first_name_key         | u               | table_with_two_unique | first_name        |                  |                   | UNIQUE (first_name)

```

Note in particular that `constraint_table` and `constraint_column` are identical for both the indvidual unique key constraints and the multi-column unique key constraint.


## SQLite

SQLite does not have catalog tables like postgres so we need a slightly different appraoch for extracting foreign key constraints. We can use [`PRAGMA`](https://www.sqlite.org/pragma.html) statements instead to query the database metadata. The `PRAGMA` statement we will use is [`PRAGMA foreign_key_list('table_name')`](https://www.sqlite.org/pragma.html#pragma_foreign_key_list) which retrieves foreign key information for a single table. Unfortunately there isn't a way to get foreign keys constraints for all tables at once so we need to do some work up front to get the list of tables and then loop over this to retrieve the foreign key constraints for each of the tables in turn.

The `PRAGMA` statement can be called using `PRAGMA foreign_key_list('table_name')` or if using `sqlite3` `SELECT * FROM pragma_foreign_key_list('table_name')`. We will use the second form as this allows us to `SELECT` the specific columns we're interested in.

Firstly we need to know our list of tables. This can be retrieved from the [`sqlite_master`](https://www.sqlite.org/faq.html#q7) table using `SELECT tbl_name FROM sqlite_master WHERE type = 'table';`.

For each returned table we now need to use the `PRAGMA` to get the foreign keys:
```
SELECT 
  'table_name' AS constraint_table,
  pragma.'from' AS constraint_column,
  pragma.'table' AS referenced_table,
  pragma.'to' AS referenced_column
FROM  pragma_foreign_key_list('table_name') AS pragma;
```

Substituting `table_name` for the name of each table in turn. Note that the quotes around `table`, `from` and `to` are required as these are protected words in SQLite so need to be quoted to be interpreted as a string literal.

Using the same setup code as above to create the three test tables we can use the query for each table and `UNION` them to get:

```
SELECT 
  'region' AS constraint_table,
  pragma.'from' AS constraint_column,
  pragma.'table' AS referenced_table,
  pragma.'to' AS referenced_column
FROM  pragma_foreign_key_list('region') AS pragma

UNION

SELECT 
  'street' AS constraint_table,
  pragma.'from' AS constraint_column,
  pragma.'table' AS referenced_table,
  pragma.'to' AS referenced_column
FROM  pragma_foreign_key_list('street') AS pragma

UNION

SELECT 
  'address' AS constraint_table,
  pragma.'from' AS constraint_column,
  pragma.'table' AS referenced_table,
  pragma.'to' AS referenced_column
FROM  pragma_foreign_key_list('address') AS pragma;

```

Running with [`.headers ON`](https://sqlite.org/cli.html#special_commands_to_sqlite3_dot_commands_) returns:
```
constraint_table|constraint_column|referenced_table|referenced_column
address|region|region|name
address|street|street|name
region|parent|region|name
```

## Bringing it all back home

We've retrieved from the database a table of the foreign key constraints which we can now use within our automatic load process. We know which table and column has the foreign key constraint on it and we know the referenced table and column. At the time of preparing the data we do not know what value any of the autogenerated columns will take once the data is added to the database but we have a unique temporary identifier to link between the different tables within R. 

To enable an automatic upload when uploading a table referenced by a foreign key constraint we strip any column of referenced temporary identifiers then insert the data returning the generated value for that column. We then loop over the tables and update all columns which reference that autogenerated column to ensure that any old temporary ID is replaced with the actual value generated during upload. Therefore when the table with the foreign key constraint on it is uploaded it points to the correct row within the referenced table.

Using this pattern means that for all data import tasks where we are just appending data to the database any upload of data can be done automatically. Beyond this if we do want to do some bespoke upload (deleting some data, or updating some rows) we can still use this bulk automatic upload and then write code around it to manage any additional requirements. Resulting in having less code to write and less code to review.
