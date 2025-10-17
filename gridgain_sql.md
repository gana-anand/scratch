# Data Manipulation Language (DML) | GridGain Documentation

# Data Manipulation Language (DML)[](#data-manipulation-language-dml)

This page includes all data manipulation language (DML) commands supported by GridGain.

## SELECT[](#select)

Retrieve data from a table or multiple tables.

```
SELECT
    [TOP term] [DISTINCT | ALL] selectExpression [,...]
    FROM tableExpression [,...] [WHERE expression]
    [GROUP BY expression [,...]] [HAVING expression]
    [{UNION [ALL] | MINUS | EXCEPT | INTERSECT} select]
    [ORDER BY order [,...]]
    [{ LIMIT expression [OFFSET expression]
    [SAMPLE_SIZE rowCountInt]} | {[OFFSET expression {ROW | ROWS}]
    [{FETCH {FIRST | NEXT} expression {ROW | ROWS} ONLY}]}]
```

### Parameters[](#parameters)

-   `DISTINCT` - removes duplicate rows from a result set.
    
-   `GROUP BY` - groups the result by the given expression(s).
    
-   `HAVING` - filters rows after grouping.
    
-   `ORDER BY` - sorts the result by the given column(s) or expression(s).
    
-   `LIMIT and FETCH FIRST/NEXT ROW(S) ONLY` - limits the number of rows returned by the query (no limit if null or smaller than zero).
    
-   `OFFSET` - specifies​ how many rows to skip.
    
-   `UNION, INTERSECT, MINUS, EXPECT` - combines the result of this query with the results of another query.
    
-   `tableExpression` - Joins a table. The join expression is not supported for cross and natural joins. A natural join is an inner join, where the condition is automatically on the columns with the same name.
    

```
tableExpression = [[LEFT | RIGHT]{OUTER}] | INNER | CROSS | NATURAL]
JOIN tableExpression
[ON expression]
```

-   `LEFT` - LEFT JOIN performs a join starting with the first (left-most) table and then any matching second (right-most) table records.
    
-   `RIGHT` - RIGHT JOIN performs a join starting with the second (right-most) table and then any matching first (left-most) table records.
    
-   `OUTER` - Outer joins subdivide further into left outer joins, right outer joins, and full outer joins, depending on which table’s rows are retained (left, right, or both).
    
-   `INNER` - An inner join requires each row in the two joined tables to have matching column values.
    
-   `CROSS` - CROSS JOIN returns the Cartesian product of rows from tables in the join.
    
-   `NATURAL` - The natural join is a special case of equi-join.
    
-   `ON` - Value or condition to join on.
    

### Description[](#description)

`SELECT` queries can be executed against both [replicated](/docs/latest/developers-guide/data-modeling/data-partitioning#replicated) and [partitioned](/docs/latest/developers-guide/data-modeling/data-partitioning#partitioned) data.

When queries are executed against fully replicated data, GridGain will send a query to a single cluster node and run it over the local data there.

On the other hand, if a query is executed over partitioned data, then the execution flow will be the following:

-   The query will be parsed and split into multiple map queries and a single reduce query.
    
-   All the map queries are executed on all the nodes where required data resides.
    
-   All the nodes provide result sets of local execution to the query initiator (reducer) that, in turn, will accomplish the reduce phase by properly merging provided result sets.
    

### JOINs[](#joins)

GridGain supports colocated and non-colocated distributed SQL joins. Furthermore, if the data resides in different tables (aka. caches in GridGain), GridGain allows for cross-table joins as well.

Joins between partitioned and replicated data sets always work without any limitations.

However, if you join partitioned data sets, then you have to make sure that the keys you are joining on are either colocated or make sure you switched on the non-colocated joins parameter for a query.

Refer to Distributed Joins page for more details.

### Group By and Order By Optimizations[](#group-by-and-order-by-optimizations)

SQL queries with `ORDER BY` clauses do not require loading the whole result set to a query initiator (reducer) node in order to complete the sorting. Instead, every node to which a query will be mapped will sort its own part of the overall result set and the reducer will do the merge in a streaming fashion.

The same optimization is implemented for sorted `GROUP BY` queries - there is no need to load the whole result set to the reducer in order to do the grouping before giving it to an application. In GridGain, partial result sets from the individual nodes can be streamed, merged, aggregated, and returned to the application gradually.

### Examples

Retrieve all rows from the `Person` table:

```
SELECT * FROM Person;
```

Get all rows in alphabetical order:

```
SELECT * FROM Person ORDER BY name;
```

Calculate the number of `Persons` from a specific city:

```
SELECT city_id, COUNT(*) FROM Person GROUP BY city_id;
```

Join data stored in the `Person` and `City` tables:

```
SELECT p.name, c.name
	FROM Person p, City c
	WHERE p.city_id = c.id;
```

## INSERT[](#insert)

Inserts data into a table.

```
INSERT INTO tableName
  {[( columnName [,...])]
  {VALUES {({DEFAULT | expression} [,...])} [,...] | [DIRECT] [SORTED] select}}
  | {SET {columnName = {DEFAULT | expression}} [,...]}
```

### Parameters[](#parameters-2)

-   `tableName` - name of the table to be updated.
    
-   `columnName` - name of a column to be initialized with a value from the VALUES clause.
    

### Description[](#description-2)

`INSERT` adds an entry or entries into a table (aka. cache in GridGain).

Since GridGain stores all the data in a form of key-value pairs, all the `INSERT` statements are finally transformed into a set of key-value operations.

If a single key-value pair is being added into a cache then, eventually, an `INSERT` statement will be converted into a `cache.putIfAbsent(…​)` operation. In other cases, when multiple key-value pairs are inserted, the DML engine creates an `EntryProcessor` for each pair and uses `cache.invokeAll(…​)` to propagate the data into a cache.

### Examples

Insert a new Person into the table:

```
INSERT INTO Person (id, name, city_id) VALUES (1, 'John Doe', 3);
```

Fill in Person table with the data retrieved from Account table:

```
INSERT INTO Person(id, name, city_id)
   (SELECT a.id + 1000, concat(a.firstName, a.secondName), a.city_id
   FROM Account a WHERE a.id > 100 AND a.id < 1000);
```

## UPDATE[](#update)

Update data in a table.

```
UPDATE tableName [[AS] newTableAlias]
  SET {{columnName = {DEFAULT | expression}} [,...]} |
  {(columnName [,...]) = (select)}
  [WHERE expression][LIMIT expression]
```

### Parameters[](#parameters-3)

-   `table` - the name of the table to be updated.
    
-   `columnName` - the name of a column to be updated with a value from a `SET` clause.
    

### Description[](#description-3)

`UPDATE` alters existing entries stored in a table.

Since GridGain stores all the data in a form of key-value pairs, all the `UPDATE` statements are finally transformed into a set of key-value operations.

Initially, the SQL engine generates and executes a `SELECT` query based on the `UPDATE WHERE` clause and only after that does it modify the existing values that satisfy the clause result.

The modification is performed via a `cache.invokeAll(…​)` operation. This means that once the result of the `SELECT` query is ready, the SQL engine will prepare a number of `EntryProcessors` and will execute all of them using a `cache.invokeAll(…​)` operation. While the data is being modified using `EntryProcessors`, additional checks are performed to make sure that nobody has interfered between the `SELECT` and the actual update.

### Primary Keys Updates[](#primary-keys-updates)

Ignite does not allow updating a primary key because the latter defines a partition the key and its value belong to statically. While the partition with all its data can change several cluster owners, the key always belongs to a single partition. The partition is calculated using a hash function applied to the key’s value.

Thus, if a key needs to be updated it has to be removed and then inserted.

### Examples

Update the `name` column of an entry:

```
UPDATE Person SET name = 'John Black' WHERE id = 2;
```

Update the `Person` table with the data taken from the `Account` table:

```
UPDATE Person p SET name = (SELECT a.first_name FROM Account a WHERE a.id = p.id)
```

## WITH[](#with)

Used to name a sub-query, can be referenced in other parts of the SQL statement.

```
WITH  { name [( columnName [,...] )] AS ( select ) [,...] }
{ select | insert | update | merge | delete | createTable }
```

### Parameters[](#parameters-4)

-   `query_name` - the name of the sub-query to be created. The name assigned to the sub-query is treated as though it was an inline view or table.
    

### Description[](#description-4)

`WITH` creates a sub-query. One or more common table entries can be referred to by name. Column name declarations are optional - the column names will be inferred from the named select queries. The final action in a WITH statement can be a `select`, `insert`, `update`, `merge`, `delete`, or `create table`.

### Example

```
WITH cte1 AS (
        SELECT 1 AS FIRST_COLUMN
), cte2 AS (
        SELECT FIRST_COLUMN+1 AS FIRST_COLUMN FROM cte1
)
SELECT sum(FIRST_COLUMN) FROM cte2;
```

## MERGE[](#merge)

Merge data into a table.

```
MERGE INTO tableName [(columnName [,...])]
  [KEY (columnName [,...])]
  {VALUES {({ DEFAULT | expression } [,...])} [,...] | select}
```

### Parameters[](#parameters-5)

-   `tableName` - the name of the table to be updated.
    
-   `columnName` - the name of a column to be initialized with a value from a `VALUES` clause.
    

### Description[](#description-5)

`MERGE` updates existing entries and inserts new entries.

Because GridGain stores all the data in a form of key-value pairs, all the `MERGE` statements are transformed into a set of key-value operations.

`MERGE` is one of the most straightforward operations because it is translated into `cache.put(…​)` and `cache.putAll(…​)` operations depending on the number of rows that need to be inserted or updated as part of the `MERGE` query.

### Examples

Merge some rows into the `Person` table:

```
MERGE INTO Person(id, name, city_id) VALUES
	(1, 'John Smith', 5),
        (2, 'Mary Jones', 5);
```

Fill in the `Person` table with the data retrieved from the `Account` table:

```
MERGE INTO Person(id, name, city_id)
   (SELECT a.id + 1000, concat(a.firstName, a.secondName), a.city_id
   FROM Account a WHERE a.id > 100 AND a.id < 1000);
```

## DELETE[](#delete)

Delete data from a table.

```
DELETE
  [TOP term] FROM tableName
  [WHERE expression]
  [LIMIT term]
```

### Parameters[](#parameters-6)

-   `tableName` - the name of the table to delete data from.
    
-   `TOP, LIMIT` - specifies the number​ of entries to be deleted (no limit if null or smaller than zero).
    

### Description[](#description-6)

`DELETE` removes data from a table (aka. cache in GridGain).

Because GridGain stores all the data in a form of key-value pairs, all the `DELETE` statements are transformed into a set of key-value operations.

A `DELETE` statements' execution is split into two phases and is similar to the execution of `UPDATE` statements.

First, using a `SELECT` query, the SQL engine gathers those keys that satisfy the `WHERE` clause in the `DELETE` statement. Next, after having all those keys in place, it creates a number of `EntryProcessors` and executes them with `cache.invokeAll(…​)`. While the data is being deleted, additional checks are performed to make sure that nobody has interfered between the `SELECT` and the actual removal of the data.

### Examples

Delete all the `Persons` with a specific name:

```
DELETE FROM Person WHERE name = 'John Doe';
```

# Transactions | GridGain Documentation

# Transactions[](#transactions)

## Description[](#description)

GridGain supports the following functions that allow you to start, commit, or rollback a transaction:

```
BEGIN [TRANSACTION]

COMMIT [TRANSACTION]

ROLLBACK [TRANSACTION]
```

A transaction is a sequence of SQL operations that starts with the `BEGIN` statement and ends with the `COMMIT` statement. Either all of the operations in a transaction succeed or they all fail.

The `ROLLBACK [TRANSACTION]` statement undoes all updates made since the last time a `COMMIT` or `ROLLBACK` command was issued.

## Example[](#example)

Add a person and update the city population by 1 in a single transaction.

```
BEGIN;

INSERT INTO Person (id, name, city_id) VALUES (1, 'John Doe', 3);

UPDATE City SET population = population + 1 WHERE id = 3;

COMMIT;
```

Roll back the changes made by the previous commands.

```
BEGIN;

INSERT INTO Person (id, name, city_id) VALUES (1, 'John Doe', 3);

UPDATE City SET population = population + 1 WHERE id = 3;
```

# Operational Commands | GridGain Documentation

# Operational Commands[](#operational-commands)

## COPY INTO[](#copy-into)

### Description[](#description)

Imports data from an external source into a table or exports data to a file from the table. The table the data is imported into must exist.

```
COPY FROM source
INTO target
FORMAT formatType
[ formatTypeOptions ]
[ PROPERTIES ( 'propName'='propValue' [,...]) ]
```

### Parameters[](#parameters)

-   `source` - the full path to the file to import the data data from (if `target` is a table). Then name of the table and the list of columns to export the data from (if `target` is a file). If the source is a table, you can further narrow down the selection by using a query.
    
-   `target` - the table and table columns when importing data to the table. The path to the file when exporting data.
    
-   `formatType` - the format of the file to work with:
    
    -   `CSV`
        
    -   `PARQUET`
        
    -   `ICEBERG`
        
    
-   `properties` - case-sensitive configuration properties:
    
    -   `s3.client-region` - when using S3 storage, the region to store the data in.
        
    -   `s3.access-key-id` - when using S3 storage, the AWS access key.
        
    -   `s3.secret-access-key` - when using S3 storage, the AWS secret key.
        
    -   For Iceberg format:
        
        -   The [catalog properties](https://iceberg.apache.org/docs/latest/configuration/#catalog-properties) are supported.
            
        -   The `table-identifier` property describes Apache Iceberg TableIdentifier names. The names can be dot-separated. For example, `db_name.table_name` or `table`.
            
        -   The Apache Iceberg `warehouse` path can be defined explicitly as a `'warehouse'='path'` property, or implicitly as a source/target `COPY FROM source INTO target`. If both ways are defined, the explicit property is used.
            
        
    
-   `formatTypeOptions` - parameters related to the selected file format:
    
    -   `delimiter` - the field delimiter for CSV files; default delimiter is `,`. Delimiter syntax is `'char'`. Any alphanumeric character can be a delimiter.
        
    -   `header` - for CSV files only; if set to `true`, specifies that the created file should contain a header line with column names. Column names of the table are used to create the header line.
        
    -   `pattern` - the file pattern used when importing partitioned Parquet tables in the regular expression format. The regular expression must be enclosed in `'` signs. For example, `'.*'` imports all files. Partitioned column will not be imported.
        
    

### Examples[](#examples)

Import data from columns `name` and `age` of a CSV file with header into Table1 columns `name` and `age`:

```
/* Import data from CSV with column headers */
COPY FROM '/path/to/dir/data.csv'
INTO Table1 (name, age)
FORMAT CSV
```

Import data from the first two columns of a CSV file without header into Table1 columns `name` and `age`:

```
/* Import data from CSV without column headers  */
COPY FROM '/path/to/dir/data.csv'
INTO Table1 (name, age)
```

Export data from Table1 to a CSV file:

```
/* Export data to CSV */
COPY FROM (SELECT name, age FROM Table1)
INTO  '/path/to/dir/data.csv'
FORMAT CSV
```

Export data from Table1 to a CSV file:

```
/* Export data to CSV */
COPY FROM Table1 (name, age)
INTO  '/path/to/dir/data.csv'
FORMAT CSV
```

Import CSV file from AWS S3 into Table1:

```
/* Import CSV file from s3 */
COPY FROM 's3://mybucket/data.csv'
INTO Table1 (name, age)
FORMAT CSV
DELIMITER '|'
PROPERTIES('s3.access-key-id' = 'keyid', 's3.secret-access-key' = 'secretkey', 's3.client-region' = 'eu-central-1')
```

A simple example of exporting data to Iceberg. For working with local file system you can use HadoopCatalog that does not need to connect to a Hive MetaStore.

```
COPY FROM Table1 (id,name,height)
INTO '/tmp/person.i/'
FORMAT ICEBERG
PROPERTIES ('table-identifier'='person', 'catalog-impl'='org.apache.iceberg.hadoop.HadoopCatalog')
```

Export data into Iceberg on AWS S3:

```
COPY FROM person (id,name,height)
INTO 's3://iceberg-warehouse/glue-catalog'
FORMAT ICEBERG
PROPERTIES(
    'table-identifier'='iceberg_db_1.person',
    'io-impl'='org.apache.iceberg.aws.s3.S3FileIO',
    'catalog-impl'='org.apache.iceberg.aws.glue.GlueCatalog',
    's3.client-region'='eu-central-1',
    's3.access-key-id'='YOUR_KEY',
    's3.secret-access-key'='YOUR_SECRET'
    )
```

Imports data from partitioned Parquet database:

```
COPY FROM '/tmp/partitioned_table_dir'
INTO city (id, name, population)
FORMAT PARQUET
PATTERN '.*'
```

Where the Parquet table looks like this:

partitioned\_table\_dir/
├─ CountryCode=USA/
│  ├─ 000000\_0.parquet
├─ CountryCode=FR/
│  ├─ 000000\_0.parquet

## SET STREAMING[](#set-streaming)

### Description[](#description-2)

Streams data in bulk to an SQL table in your GridGain cluster. When streaming is enabled, the JDBC/ODBC driver packs your commands in batches and sends them to the server (GridGain cluster). On the server side, the batch is converted into a stream of cache update commands, which are distributed asynchronously between the server nodes. Performing this asynchronously increases peak throughput because at any given time all cluster nodes are busy with data loading.

```
SET STREAMING [OFF|ON];
```

### Usage[](#usage)

To stream data into your GridGain cluster, prepare a file with the `SET STREAMING ON` command followed by `INSERT` commands for the data to be loaded. For example:

```
SET STREAMING ON;

INSERT INTO City(ID, Name, CountryCode, District, Population) VALUES (1,'Kabul','AFG','Kabol',1780000);
INSERT INTO City(ID, Name, CountryCode, District, Population) VALUES (2,'Qandahar','AFG','Qandahar',237500);
INSERT INTO City(ID, Name, CountryCode, District, Population) VALUES (3,'Herat','AFG','Herat',186800);
INSERT INTO City(ID, Name, CountryCode, District, Population) VALUES (4,'Mazar-e-Sharif','AFG','Balkh',127800);
INSERT INTO City(ID, Name, CountryCode, District, Population) VALUES (5,'Amsterdam','NLD','Noord-Holland',731200);
-- More INSERT commands --
```

```
CREATE TABLE City (
  ID INT(11),
  Name CHAR(35),
  CountryCode CHAR(3),
  District CHAR(20),
  Population INT(11),
  PRIMARY KEY (ID, CountryCode)
) WITH "template=partitioned, backups=1, affinityKey=CountryCode, CACHE_NAME=City, KEY_TYPE=demo.model.CityKey, VALUE_TYPE=demo.model.City";

SET STREAMING ON;

INSERT INTO City(ID, Name, CountryCode, District, Population) VALUES (1,'Kabul','AFG','Kabol',1780000);
INSERT INTO City(ID, Name, CountryCode, District, Population) VALUES (2,'Qandahar','AFG','Qandahar',237500);
INSERT INTO City(ID, Name, CountryCode, District, Population) VALUES (3,'Herat','AFG','Herat',186800);
INSERT INTO City(ID, Name, CountryCode, District, Population) VALUES (4,'Mazar-e-Sharif','AFG','Balkh',127800);
INSERT INTO City(ID, Name, CountryCode, District, Population) VALUES (5,'Amsterdam','NLD','Noord-Holland',731200);
-- More INSERT commands --
```

### Known Limitations[](#known-limitations)

While streaming mode allows you to load data much faster than other data loading techniques, it has some limitations:

-   Only `INSERT` commands are allowed; any attempt to execute `SELECT` or any other DML or DDL command will cause an exception.
    
-   Due to streaming mode’s asynchronous nature, you cannot know update counts for every statement executed; all JDBC/ODBC commands returning update counts will return 0.
    

### Example[](#example)

Use the world.sql file that is shipped with the latest GridGain CE distribution. It can be found in the `{gridgain_dir}/examples/sql/` directory. You can use the `run` command from [SQLLine](/docs/latest/administrators-guide/tools-analytics/sqlline), as shown below:

```
!run /apache_ignite_version/examples/sql/world.sql
```

After executing the above command and **closing the JDBC connection**, all data will be loaded into the cluster and ready to be queried.

![set streaming](/docs/8.9.25/images/set-streaming.png)

## KILL QUERY[](#kill-query)

### Description[](#description-3)

Cancel a running query. When a query is canceled with the `KILL` command, all parts of the query running on all other nodes are canceled too.

```
KILL QUERY {ASYNC} '<query_id>'
```

### Parameters[](#parameters-2)

-   `ASYNC` - an optional parameter, which returns control immediately without waiting for the cancellation to finish
    
-   `query_id` - retrieved via the SQL\_QUERIES [system view](/docs/latest/administrators-guide/monitoring-metrics/system-views#sql_queries)
    

## KILL CONTINUOUS[](#kill-continuous)

### Description[](#description-4)

Cancel a running continuous query.

```
KILL CONTINUOUS '<node_id>' '<routine_id>'
```

### Parameters[](#parameters-3)

-   `node_id` - the id of the source node of the continuous query. Retrieved via the CONTINUOUS\_QUERIES [system view](/docs/latest/administrators-guide/monitoring-metrics/system-views#continuous_queries).
    
-   `routine_id` - the continuous query ID. retrieved via the CONTINUOUS\_QUERIES [system view](/docs/latest/administrators-guide/monitoring-metrics/system-views#continuous_queries).
    

## COPY[](#copy)

Copy data from a CSV file into a SQL table.

```
COPY FROM '/path/to/local/file.csv'
INTO tableName (columnName, columnName, ...) FORMAT CSV [CHARSET '<charset-name>']
```

### Parameters[](#parameters-4)

-   `'/path/to/local/file.csv'` - the actual path to the CSV file
    
-   `tableName` - the name of the table to copy the data to
    
-   `columnName` - the name of a column corresponding to a column in the CSV file
    

### Example[](#example-2)

```
COPY FROM '/path/to/local/file.csv' INTO city (
  ID, Name, CountryCode, District, Population) FORMAT CSV
```

In the above command, substitute `/path/to/local/file.csv` with the actual path to your CSV file. For instance, you can use `city.csv` that is shipped with the latest GridGain CE distribution. You can find it in your `{gridgain_dir}/examples/src/main/resources/sql/` directory.

# Aggregate Functions | GridGain Documentation

# Aggregate Functions[](#aggregate-functions)

## AVG[](#avg)

### Description[](#description)

Computes and returns the average (mean) value of the specified parameter for the selected table rows. If no rows are selected, the result is `NULL`. The returned value is of the same data type as the specified parameter.

```
AVG ([DISTINCT] expression)
```

### Parameters[](#parameters)

-   `expression` - any valid numeric expression
    
-   `DISTINCT` - an optional keyword; if present, the function averages unique values only
    

### Example[](#example)

Calculate the average players' age:

```
SELECT AVG(age) "AverageAge" FROM Players;
```

## BIT\_AND[](#bit_and)

### Description[](#description-2)

Returns the bitwise AND of all non-null values. If no rows are selected, the result is `NULL`. A logical AND operation is performed on each pair of corresponding bits of two binary expressions of equal length. In each pair, it returns 1 if the first bit is 1 AND the second bit is 1. Else, it returns 0.

```
BIT_AND (expression)
```

### Parameters[](#parameters-2)

`expression` - the input data

## BIT\_OR[](#bit_or)

### Description[](#description-3)

Return the bitwise OR of all non-null values. If no rows are selected, the result is `NULL`. A logical OR operation is performed on each pair of corresponding bits of two binary expressions of equal length. In each pair, the result is 1 if the first bit is 1 OR the second bit is 1 OR both bits are 1. Otherwise, the result is 0.

```
BIT_OR (expression)
```

### Parameters[](#parameters-3)

`expression` - the input data

## COUNT[](#count)

### Description[](#description-4)

Counts all entries or all non-null values. Returns a long. If no entries are selected, the result is `0`.

```
COUNT (* | [DISTINCT] expression)
```

### Example[](#example-2)

Calculate the number of players in each city:

```
SELECT city_id, COUNT(*) FROM Players GROUP BY city_id;
```

## FIRSTVALUE[](#firstvalue)

### Description[](#description-5)

Returns the value of `expression1` associated with the smallest value of `expression2` for each group defined by the `group by` expression in the query.

```
FIRSTVALUE ([DISTINCT] <expression1>, <expression2>)
```

This function can only be used with colocated data; you have to use the `colocated` flag when executing the query.

The collocated hint can be set as follows:

-   `SqlFieldsQuery.collocated = true` if you use [GridGain SQL API](/docs/latest/developers-guide/SQL/sql-api) to execute queries
    
-   [JDBC connection string parameter](/docs/latest/developers-guide/SQL/JDBC/jdbc-driver#parameters)
    
-   [ODBC connection string argument](/docs/latest/developers-guide/SQL/ODBC/connection-string-dsn#supported-arguments)
    

### Example[](#example-3)

Return the youngest person for each Company from the Person table and group them by Company ID:

```
select company_id, firstvalue(name, age) as youngest from person group by company_id;
```

## GROUP\_CONCAT[](#group_concat)

### Description[](#description-6)

Concatenates strings with a separator. The default separator is ',' (without whitespace). Returns a string. If no entries are selected, the result is `NULL`.

```
GROUP_CONCAT([DISTINCT] expression || [expression || [expression ...]]
  [ORDER BY expression [ASC|DESC], [[ORDER BY expression [ASC|DESC]]]
  [SEPARATOR expression])
```

`Expression` can be a concatenation of columns and strings with the `||` operator; for example: `column1 || "=" || column2`.

### Parameters[](#parameters-4)

-   `DISTINCT` - filters the result set for unique sets of expressions
    
-   `expression` - specifies an expression that may be a column name, a result of another function, or a math operation
    
-   `ORDER BY` - orders rows by expression
    
-   `SEPARATOR` - overrides a string separator; by default, the separator character is the comma ','
    

### Example[](#example-4)

Group all players' names in one row:

```
SELECT GROUP_CONCAT(name ORDER BY id SEPARATOR ', ') FROM Players;
```

## LASTVALUE[](#lastvalue)

### Description[](#description-7)

Returns the value of `expression1` associated with the largest value of `expression2` for each group defined by the `group by` expression.

```
LASTVALUE ([DISTINCT] <expression1>, <expression2>)
```

This function can only be used with colocated data and you have to use the `collocated` flag when executing the query.

The colocated hint can be set as follows:

-   `SqlFieldsQuery.collocated=true` if you use [GridGain SQL API](/docs/latest/developers-guide/SQL/sql-api) to execute queries
    
-   [JDBC connection string parameter](/docs/latest/developers-guide/SQL/JDBC/jdbc-driver#parameters)
    
-   [ODBC connection string argument](/docs/latest/developers-guide/SQL/ODBC/connection-string-dsn#supported-arguments)
    

### Example[](#example-5)

Return the oldest person for each Company from the Person table and group them by Company ID:

```
select company_id, lastvalue(name, age) as oldest from person group by company_id;
```

## MAX[](#max)

### Description[](#description-8)

Returns the highest value for the specified parameter. If no entries are selected, the result is `NULL`. The returned value is of the same data type as the parameter.

```
MAX (expression)
```

### Parameters[](#parameters-5)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-6)

Return the height of the tallest player:

```
SELECT MAX(height) FROM Players;
```

## MIN[](#min)

### Description[](#description-9)

Returns the lowest value for the specified parameter. If no entries are selected, the result is `NULL`. The returned value is of the same data type as the parameter.

```
MIN (expression)
```

### Parameters[](#parameters-6)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-7)

Return the age of the youngest player:

```
SELECT MIN(age) FROM Players;
```

## SUM[](#sum)

### Description[](#description-10)

Returns the sum of all values of a parameter. If no entries are selected, the result is `NULL`. The data type of the returned value depends on the parameter data type.

```
SUM ([DISTINCT] expression)
```

### Parameters[](#parameters-7)

-   `DISTINCT` - sum up unique values only
    
-   `expression` - may be a column name, a result of another function, or a math operation
    

### Example[](#example-8)

Get the total number of goals scored by all players:

```
SELECT SUM(goal) FROM Players;
```

## STDDEV\_POP[](#stddev_pop)

### Description[](#description-11)

Returns the standard deviation value for a population (`double`). If no entries are selected, the result is `NULL`.

```
STDDEV_POP ([DISTINCT] expression)
```

### Parameters[](#parameters-8)

-   `DISTINCT` - calculate for unique values only
    
-   `expression` - may be a column name, etc.; see [Microsoft language manual](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/functions/stddev_pop)
    

### Example[](#example-9)

Calculate the standard deviation for players' age:

```
SELECT STDDEV_POP(age) from Players;
```

## STDDEV\_SAMP[](#stddev_samp)

### Description[](#description-12)

Calculates the standard deviation for a sample (`double`). If no entries are selected, the result is `NULL`.

```
STDDEV_SAMP ([DISTINCT] expression)
```

### Parameters[](#parameters-9)

-   `DISTINCT` - calculate for unique values only
    
-   `expression` - may be a column name, etc.; see [Microsoft language manual](https://stackoverflow.com/questions/62660321/stddev-pop-vs-stddev-samp)
    

### Example[](#example-10)

Calculate the sample standard deviation for players' age:

```
SELECT STDDEV_SAMP(age) from Players;
```

## VAR\_POP[](#var_pop)

### Description[](#description-13)

Calculates the _variance_ (square of the standard deviation) for a population. It returns a `double`. If no entries are selected, the result is `NULL`.

```
VAR_POP ([DISTINCT] expression)
```

### Parameters[](#parameters-10)

-   `DISTINCT` - calculate for unique values only
    
-   `expression` - may be a column name, etc.; see [Microsoft language manual](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/functions/var_pop)
    

### Example[](#example-11)

Calculate the variance of players' age:

```
SELECT VAR_POP (age) from Players;
```

## VAR\_SAMP[](#var_samp)

### Description[](#description-14)

Calculates the _variance_ (square of the standard deviation) for a sample. It returns a `double`. If no entries are selected, the result is `NULL`.

```
VAR_SAMP ([DISTINCT] expression)
```

### Parameters[](#parameters-11)

-   `DISTINCT` - calculate for unique values only
    
-   `expression` - may be a column name, etc.; see [Microsoft language manual](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/functions/var_samp)
    

### Example[](#example-12)

Calculate the variance of players' age:

```
SELECT VAR_SAMP(age) FROM Players;
```

# JSON Functions | GridGain Documentation

# JSON Functions[](#json-functions)

## IS\_JSON[](#is_json)

### Description[](#description)

Checks if the string contains valid JSON content.

```
IS_JSON ( string [, json_type_constraint] )
```

### Parameters[](#parameters)

-   `string` - the string to check
    
-   `json_type_constraint` - the JSON content type to check for; possible values:
    
    -   `VALUE`
        
    -   `ARRAY`
        
    -   `OBJECT`
        
    -   `SCALAR`
        
    

### Example[](#example)

Check 'years' for JSON objects:

```
SELECT IS_JSON('{"years":[1999, 2011, 2022]}');
```

## JSON\_ARRAY[](#json_array)

### Description[](#description-2)

Creates a JSON array from the specified expressions.

```
JSON_ARRAY ( [ <json_array_value> [,...n] ])
```

### Parameters[](#parameters-2)

`json_array_value` - the value of an element in the JSON array

### Example[](#example-2)

Create a JSON array out of the elements: 'example', 1, and 4.2:

```
SELECT JSON_ARRAY('example', 1, 4.2)
```

## JSON\_MODIFY[](#json_modify)

### Description[](#description-3)

Updates the value of a property and returns the updated JSON string.

```
JSON_MODIFY ( expression , json_path , newValue )
```

### Parameters[](#parameters-3)

-   `expression` - the name of a variable or a column that contains JSON text
    
-   `path` - JSON path that specifies an object or an array to extract
    
-   `newValue` - the new value to assign to the specified property
    

### Example[](#example-3)

Change Bristol to London in the 'info' JSON string.

```
//Initial JSON
//{"info":{"type":1,"address":{"town":"Bristol","country":"England"},"tags":["Sport","Water polo"]}}


SELECT JSON_MODIFY(J, '$.info.address.town', 'London') FROM TEST;

//Updated JSON
//{"info":{"type":1,"address":{"town":"London","country":"England"},"tags":["Sport","Water polo"]}}
```

## JSON\_OBJECT[](#json_object)

### Description[](#description-4)

Creates a JSON object based on the specified expression.

```
JSON_OBJECT ( [ <json_key_value> [,...n] ])
```

### Parameters[](#parameters-4)

json\_key\_value\` - an expression that defines the value of the JSON key: `(key1, value1, key2, value2…​)`

### Example[](#example-4)

Create a JSON object:

```
select JSON_OBJECT(SELECT * FROM (select * from VALUES (1), (2), (3), (4), (5), (6), (7)) as t limit 4)
```

## JSON\_QUERY[](#json_query)

### Description[](#description-5)

Extracts an object or an array from a JSON string.

```
JSON_QUERY ( expression , path )
```

### Parameters[](#parameters-5)

-   `expression` - the name of a variable or a column that contains JSON text
    
-   `path` - JSON path that specifies the object or array to extract
    

### Example[](#example-5)

Extract the 'info' object:

```
select JSON_QUERY('{"info":{"address":[{"town":"Paris"},{"town":"London"}]}}','$.info.address')
```

## JSON\_VALUE[](#json_value)

### Description[](#description-6)

Extracts a scalar value from a JSON string.

```
JSON_VALUE( expression , path )
```

### Parameters[](#parameters-6)

-   `expression` - the name of a variable or a column that contains JSON text
    
-   `path` - JSON path that specifies the value to extract
    

### Example[](#example-6)

Extract the 'town' value from the 'info' JSON string.

```
select JSON_VALUE('{"info":{"address":[{"town":"Paris"},{"town":"London"}]}}','$.info.address[0].town')
```

# Numeric Functions | GridGain Documentation

# Numeric Functions[](#numeric-functions)

## ABS[](#abs)

### Description[](#description)

Returns the absolute value of an expression.

```
ABS (expression)
```

### Parameters[](#parameters)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example)

Calculate an absolute value:

```
SELECT transfer_id, ABS (price) from Transfers;
```

## ACOS[](#acos)

### Description[](#description-2)

Calculates the arc cosine, returns a `double`.

```
ACOS (expression)
```

### Parameters[](#parameters-2)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-2)

Calculate the arc cos value:

```
SELECT acos(angle) FROM Triangles;
```

## ASIN[](#asin)

### Description[](#description-3)

Calculates the arc sine, returns a `double`.

```
ASIN (expression)
```

### Parameters[](#parameters-3)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-3)

Calculate an arc sine:

```
SELECT asin(angle) FROM Triangles;
```

## ATAN[](#atan)

### Description[](#description-4)

Calculates the arc tangent, returns a `double`.

```
ATAN (expression)
```

### Parameters[](#parameters-4)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-4)

Calculate an arc tangent:

```
SELECT atan(angle) FROM Triangles;
```

## COS[](#cos)

### Description[](#description-5)

Calculates the trigonometric cosine, returns a `double`.

```
COS (expression)
```

### Parameters[](#parameters-5)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-5)

Calculate a cosine:

```
SELECT COS(angle) FROM Triangles;
```

## COSH[](#cosh)

### Description[](#description-6)

Calculates the hyperbolic cosine, returns a `double`.

```
COSH (expression)
```

### Parameters[](#parameters-6)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-6)

Calculate n hyperbolic cosine:

```
SELECT HCOS(angle) FROM Triangles;
```

## COT[](#cot)

### Description[](#description-7)

Calculates the trigonometric cotangent (1/TAN(ANGLE)), returns a `double`.

```
COT (expression)
```

### Parameters[](#parameters-7)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-7)

Calculate trigonometric cotangent:

```
SELECT COT(angle) FROM Triangles;
```

## SIN[](#sin)

### Description[](#description-8)

Calculates the trigonometric sine, returns a `double`.

```
SIN (expression)
```

### Parameters[](#parameters-8)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-8)

Calculate a trigonometric sine:

```
SELECT SIN(angle) FROM Triangles;
```

## SINH[](#sinh)

### Description[](#description-9)

Calculates the hyperbolic sine, returns a `double`.

```
SINH (expression)
```

### Parameters[](#parameters-9)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-9)

Calculate a hyperbolic sine:

```
SELECT SINH(angle) FROM Triangles;
```

## TAN[](#tan)

### Description[](#description-10)

Calculates the trigonometric tangent, returns a `double`.

```
TAN (expression)
```

### Parameters[](#parameters-10)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-10)

Calculate a trigonometric tangent:

```
SELECT TAN(angle) FROM Triangles;
```

## TANH[](#tanh)

### Description[](#description-11)

Calculates the hyperbolic tangent, returns a `double`.

```
TANH (expression)
```

### Parameters[](#parameters-11)

`expression` - may be a column name, a result of another function, or a math operation

### Example[](#example-11)

Calculate a hyperbolic tangent:

```
SELECT TANH(angle) FROM Triangles;
```

## ATAN2[](#atan2)

### Description[](#description-12)

Calculates the angle when converting the rectangular coordinates to the polar coordinates, returns a `double`.

```
ATAN2 (y, x)
```

### Parameters[](#parameters-12)

`x` and `y` - the arguments

### Example[](#example-12)

Calculate a 2-argument arctangent:

```
SELECT ATAN2(X, Y) FROM Triangles;
```

## BITAND[](#bitand)

### Description[](#description-13)

The bitwise AND operation, returns a `long`.

```
BITAND (y, x)
```

### Parameters[](#parameters-13)

`x` and `y` - the arguments.

### Example[](#example-13)

Return the bitwise AND:

```
SELECT BITAND(X, Y) FROM Triangles;
```

## BITGET[](#bitget)

### Description[](#description-14)

Returns `true` if and only if the first parameter has a bit set in the position specified by the second parameter. The second parameter is zero-indexed; the least significant bit has position 0.

```
BITGET (y, x)
```

### Parameters[](#parameters-14)

`x` and `y` - the arguments

### Example[](#example-14)

Verify that teh third bit is 1:

```
SELECT BITGET(X, 3) from Triangles;
```

## BITOR[](#bitor)

### Description[](#description-15)

The bitwise OR operation, returns a `long`.

```
BITOR (y, x)
```

### Parameters[](#parameters-15)

`x` and `y` - the arguments

### Example[](#example-15)

Perform the bitwise OR operation:

```
SELECT BITGET(X, Y) from Triangles;
```

## BITXOR[](#bitxor)

### Description[](#description-16)

The bitwise XOR operation, returns a `long`.

```
BITXOR (y, x)
```

### Parameters[](#parameters-16)

`x` and `y` - the arguments

### Example[](#example-16)

Perform the bitwise XOR operation:

```
SELECT BITXOR(X, Y) FROM Triangles;
```

## MOD[](#mod)

### Description[](#description-17)

The modulo operation, returns a `long`.

```
MOD (y, x)
```

### Parameters[](#parameters-17)

`x` and `y` - the arguments

### Example[](#example-17)

Calculate MOD between two fields:

```
SELECT BITXOR(X, Y) FROM Triangles;
```

## CEILING[](#ceiling)

### Description[](#description-18)

Returns the smallest integer value that is greater than, or equal to, the argument. The returned value is of the same type as the argument, with the scale set to `0` the precision adjusted (if applicable).

```
CEIL (expression)
CEILING (expression)
```

### Parameters[](#parameters-18)

`expression` - any valid numeric expression

### Example[](#example-18)

Calculate a ceiling price for items:

```
SELECT item_id, CEILING(price) FROM Items;
```

## DEGREES[](#degrees)

### Description[](#description-19)

Converts an angle measured in radians to an approximately equivalent angle measured in degrees, returns a `double`.

```
DEGREES (expression)
```

### Parameters[](#parameters-19)

`expression` - any valid numeric expression

### Example[](#example-19)

Convert radians to degrees:

```
SELECT DEGREES(X) FROM Triangles;
```

## EXP[](#exp)

### Description[](#description-20)

Calculates E raised to the power of x, returns a `double`.

```
EXP (expression)
```

### Parameters[](#parameters-20)

`expression` - any valid numeric expression

### Example[](#example-20)

Calculate exp(X):

```
SELECT EXP(X) FROM Triangles;
```

## FLOOR[](#floor)

### Description[](#description-21)

Returns the largest integer value that is less than, or equal to, the argument. The returned value is of the same type as the argument, with the scale set to `0` the precision adjusted (if applicable).

```
FLOOR (expression)
```

### Parameters[](#parameters-21)

`expression` - any valid numeric expression

### Example[](#example-21)

Calculate a floor price:

```
SELECT FLOOR(X) FROM Items;
```

## LOG[](#log)

### Description[](#description-22)

Calculates the natural logarithm (base e) of a `double` value, returns a `double`.

```
LOG (expression)
LN (expression)
```

### Parameters[](#parameters-22)

`expression` - any valid numeric expression

### Example[](#example-22)

Calculate a natural logarithm:

```
SELECT LOG(X) from Items;
```

## LOG10[](#log10)

### Description[](#description-23)

Calculates the base 10 logarithm of a `double` value, returns a `double`.

```
LOG10 (expression)
```

### Parameters[](#parameters-23)

`expression` - any valid numeric expression

### Example[](#example-23)

Calculate a base 10 algorithm:

```
SELECT LOG(X) FROM Items;
```

## RADIANS[](#radians)

## Description[](#description-24)

Converts an angle measured in degrees to an approximately equivalent angle measured in radians, returns a `double`.

```
RADIANS (expression)
```

### Parameters[](#parameters-24)

`expression` - any valid numeric expression

### Example[](#example-24)

Calculate RADIANS:

```
SELECT RADIANS(X) FROM Items;
```

## SQRT[](#sqrt)

### Description[](#description-25)

Calculates the correctly rounded positive square root of a double value, returns a `double`.

```
SQRT (expression)
```

### Parameters[](#parameters-25)

`expression` - any valid numeric expression

### Example[](#example-25)

Calculate a square root:

```
SELECT SQRT(X) FROM Items;
```

## PI[](#pi)

## Description[](#description-26)

Returns Pi as a static final `double` constant.

```
PI (expression)
```

### Example[](#example-26)

Calculate Pi:

```
SELECT PI(X) FROM Items;
```

## POWER[](#power)

### Description[](#description-27)

Calculates a number raised to the power of some other number, returns a `double`.

```
POWER (X, Y)
```

### Parameters[](#parameters-26)

-   `x` - the base
    
-   `y` - the power
    

### Example[](#example-27)

Calculate n in the power of 2:

```
SELECT pow(n, 2) FROM Rows;
```

## RAND[](#rand)

### Description[](#description-28)

When called without a parameter, returns a pseudo random number. When called with a parameter, seeds the session’s random number generator. Returns a `double` between 0 (including) and 1 (excluding).

```
{RAND | RANDOM} ([expression])
```

### Parameters[](#parameters-27)

`expression` - any valid numeric expression

### Example[](#example-28)

Return a random number for every play:

```
SELECT random() FROM Play;
```

## RANDOM\_UUID[](#random_uuid)

### Description[](#description-29)

Returns a new UUID with 122 pseudo random bits.

```
{RANDOM_UUID | UUID} ()
```

### Example[](#example-29)

Return a random number for every Player:

```
SELECT UUID(),name FROM Player;
```

## ROUND[](#round)

### Description[](#description-30)

Rounds to the specified number of digits, or to the nearest long (if the number of digits if not specified). Returns a `numeric` (the same type as the input).

```
ROUND ( expression [, precision] )
```

### Parameters[](#parameters-28)

-   `expression` - any valid numeric expression
    
-   `precision` - the number of digits after the decimal point to round to
    

### Example[](#example-30)

Convert every Player’s age to an integer:

```
SELECT name, ROUND(age) FROM Player;
```

## ROUNDMAGIC[](#roundmagic)

### Description[](#description-31)

Rounds numbers. Has a special handling algorithm for numbers around 0. Only numbers smaller than or equal to `+/-1000000000000` are supported. The value is converted to a `string` internally, and then the last 4 characters are checked. '000x' becomes '0000' and '999x' becomes '999999', which is rounded automatically. This function returns a `double`.

```
ROUNDMAGIC (expression)
```

### Parameters[](#parameters-29)

`expression` - any valid numeric expression

### Example[](#example-31)

Round every Player’s age:

```
SELECT name, ROUNDMAGIC(AGE/3*3) FROM Player;
```

## SECURE\_RAND[](#secure_rand)

### Description[](#description-32)

Generates a number of cryptographically secure random numbers, returns `bytes`.

```
SECURE_RAND (int)
```

### Parameters[](#parameters-30)

`int` - the number of digits

### Example[](#example-32)

Get a truly random number:

```
SELECT name, SECURE_RAND(10) FROM Player;
```

## SIGN[](#sign)

### Description[](#description-33)

Returns `-1` if the value is smaller than zero, `0` if zero, `1` otherwise.

```
SIGN (expression)
```

### Parameters[](#parameters-31)

`expression` - any valid numeric expression

### Example[](#example-33)

Get a sign for every value:

```
SELECT name, SIGN(VALUE) FROM Player;
```

## ENCRYPT[](#encrypt)

### Description[](#description-34)

Encrypts data using a key. The supported algorithm is AES. The block size is 16 bytes. Returns `bytes`.

```
ENCRYPT (algorithmString , keyBytes , dataBytes)
```

### Parameters[](#parameters-32)

-   `algorithmString` - the AES algorithm
    
-   `keyBytes` - the key
    
-   `dataBytes` - data block size
    

### Example[](#example-34)

Encrypt players names:

```
SELECT ENCRYPT('AES', '00', STRINGTOUTF8(Name)) FROM Player;
```

## DECRYPT[](#decrypt)

### Description[](#description-35)

Decrypts data using a key. The supported algorithm is AES. The block size is 16 bytes. Returns `bytes`.

```
DECRYPT (algorithmString , keyBytes , dataBytes)
```

### Parameters[](#parameters-33)

-   `algorithmString` - the AES algorithm
    
-   `keyBytes` - the key
    
-   `dataBytes` - data block size
    

### Example[](#example-35)

Decrypt Players' names:

```
SELECT DECRYPT('AES', '00', '3fabb4de8f1ee2e97d7793bab2db1116'))) FROM Player;
```

## TRUNCATE[](#truncate)

### Description[](#description-36)

Truncates to a number of digits (to the next value closer to 0). Returns a `double`. When used with a timestamp, truncates the timestamp to the date (day) value. When used with a date, truncates the date to the date (day) value less the time part. When used with a timestamp as `string`, truncates the timestamp to a date (day) value.

```
{TRUNC | TRUNCATE} (\{\{numeric, digitsInt} | timestamp | date | timestampString})
```

### Parameters[](#parameters-34)

-   `digitsInt` - the number for digits to truncate to
    
-   `timestamp` - the timestamp to truncate
    
-   `date` - the date to truncate to
    
-   `timestampString` - the timestamp expressed as a string
    

### Example[](#example-36)

Truncate teg value to 2 digits:

```
TRUNCATE(VALUE, 2);
```

## COMPRESS[](#compress)

### Description[](#description-37)

Compresses the data using the specified compression algorithm. The supported algorithms are: LZF (faster but lower compression; default) and DEFLATE (higher degree of compression). Compression does not always reduce size. Very small objects and objects with little redundancy may get larger. This function returns `bytes`.

```
COMPRESS(dataBytes [, algorithmString])
```

### Parameters[](#parameters-35)

-   `dataBytes` - the data to compress
    
-   `algorithmString` - the algorithm to use for compression
    

### Example[](#example-37)

Compress STRINGTOUTF8 using the LZF (default) algorithm:

```
COMPRESS(STRINGTOUTF8('Test'))
```

## EXPAND[](#expand)

### Description[](#description-38)

Expands data that was previously compressed using the the COMPRESS function, returns `bytes`.

```
EXPAND(dataBytes)
```

### Parameters[](#parameters-36)

`dataBytes` - the data to expand

### Example[](#example-38)

Converts the string to UTF8 format, compress it, expand it, and converts it back to Unicode:

```
UTF8TOSTRING(EXPAND(COMPRESS(STRINGTOUTF8('LZF'))))
```

## ZERO[](#zero)

### Description[](#description-39)

Returns the value of `0`. Can be used even if numeric literals are disabled.

```
ZERO()
```

### Example[](#example-39)

Return 0:

```
ZERO()
```

# String Functions | GridGain Documentation

# String Functions[](#string-functions)

## ASCII[](#ascii)

### Description[](#description)

Returns the ASCII value of the first character in the string as an `int`.

```
ASCII(string)
```

### Parameters[](#parameters)

`string` - the argument

### Example[](#example)

Return the ASCII for the first character in the Players' names:

```
select ASCII(name) FROM Players;
```

## BIT\_LENGTH[](#bit_length)

### Description[](#description-2)

Returns the number of bits in a string as a `long`. For `BLOB`, `CLOB`, `BYTES`, and `JAVA_OBJECT`, the object’s specified precision is used. Each character needs 16 bits.

```
BIT_LENGTH(string)
```

### Parameters[](#parameters-2)

`string` - the argument

### Example[](#example-2)

Return the bit length for Players' names:

```
select BIT_LENGTH(name) FROM Players;
```

## LENGTH[](#length)

### Description[](#description-3)

Returns the number of characters in a string as a `long`. For `BLOB`, `CLOB`, `BYTES`, and `JAVA_OBJECT`, the object’s specified precision is used.

```
{LENGTH | CHAR_LENGTH | CHARACTER_LENGTH} (string)
```

### Parameters:[](#parameters-3)

`string` - the argument

### Example[](#example-3)

Return the length of Players' names:

```
SELECT LENGTH(name) FROM Players;
```

## OCTET\_LENGTH[](#octet_length)

### Description[](#description-4)

Returns the number of bytes in a string as a `long`. For `BLOB`, `CLOB`, `BYTES` and `JAVA_OBJECT`, the object’s specified precision is used. Each character needs 2 bytes.

```
OCTET_LENGTH(string)
```

### Parameters[](#parameters-4)

`string` - the argument

### Example[](#example-4)

Return the number of bytes in Players' names:

```
SELECT OCTET_LENGTH(name) FROM Players;
```

## CHAR[](#char)

### Description[](#description-5)

Returns the character that represents the ASCII value of an integer as a `string`.

```
{CHAR | CHR} (int)
```

### Parameters[](#parameters-5)

`int` - the argument

### Example[](#example-5)

Return the ASCII representation from Players' names:

```
SELECT CHAR(65)||name FROM Players;
```

## CONCAT[](#concat)

### Description[](#description-6)

Concatenates strings. Unlike with the `||` operator, NULL parameters are ignored and do not cause the result to become NULL. This function returns a `string`.

```
CONCAT(string, string [,...])
```

### Parameters[](#parameters-6)

`string` - the argument

### Example[](#example-6)

Concatenate Players' names:

```
SELECT CONCAT(NAME, '!') FROM Players;
```

## CONCAT\_WS[](#concat_ws)

### Description[](#description-7)

Concatenates strings and inserts a separator. Unlike with the `||` operator, NUL parameters are ignored, and do not cause the result to become NULL. This function returns a `string`.

```
CONCAT_WS(separatorString, string, string [,...])
```

### Parameters[](#parameters-7)

-   `separatorString` - the separator to insert
    
-   `string` - the argument
    

### Example[](#example-7)

Concatenate Players' names with comma as a separator:

```
SELECT CONCAT_WS(',', NAME, '!') FROM Players;
```

## DIFFERENCE[](#difference)

### Description[](#description-8)

Returns the difference between the `SOUNDEX()` values of two strings as an\`int\`.

```
DIFFERENCE(X, Y)
```

### Parameters[](#parameters-8)

`X` and `Y` - strings to compare

### Example[](#example-8)

Calculate the SOUNDEX() difference for two Players' names:

```
select DIFFERENCE(T1.NAME, T2.NAME) FROM players T1, players T2
   WHERE T1.ID = 10 AND T2.ID = 11;
```

## HEXTORAW[](#hextoraw)

### Description[](#description-9)

Converts a hex representation of a string to a `string`. Uses 4 hex characters per string character.

```
HEXTORAW(string)
```

### Parameters[](#parameters-9)

`string` - a hex string to convert

### Example[](#example-9)

Return a string representation of Players' names:

```
SELECT HEXTORAW(DATA) FROM Players;
```

## RAWTOHEX[](#rawtohex)

### Description[](#description-10)

Converts a string to the hex representation. Uses 4 hex characters per string character. Returns a `string`.

```
RAWTOHEX(string)
```

### Parameters[](#parameters-10)

`string` - a string to convert

### Example[](#example-10)

Return hex representations of Players' names:

```
SELECT RAWTOHEX(DATA) FROM Players;
```

## INSTR[](#instr)

### Description[](#description-11)

Returns the location of a search string in a string. If a start position is used, the characters before it are ignored. If the position is negative, the rightmost location is returned. Zero (0) is returned if the search string is not found.

```
INSTR(string, searchString, [, startInt])
```

### Parameters[](#parameters-11)

-   `string` - the string to search in
    
-   `searchString` - the string to search for
    
-   `startInt` - start position for the search
    

### Example[](#example-11)

Check if a string includes "@":

```
SELECT INSTR(EMAIL,'@') FROM Players;
```

## INSERT[](#insert)

### Description[](#description-12)

Inserts an additional string into the original string at a specified start position, returns a `string`.

```
INSERT(originalString, startInt, lengthInt, addString)
```

### Parameters:[](#parameters-12)

-   `originalString` - the original string
    
-   `startInt` - the start position
    
-   `lengthInt` - the number of characters to remove at the start position in the original string
    
-   `addString` - the string to insert
    

### Example[](#example-12)

Insert a blank space at the beginning of Players' names.

```
SELECT INSERT(NAME, 1, 1, ' ') FROM Players;
```

## LOWER[](#lower)

### Description[](#description-13)

Converts a string to lowercase.

```
{LOWER | LCASE} (string)
```

### Parameters[](#parameters-13)

`string` - the string to convert

### Example[](#example-13)

Convert to lower case Players' names:

```
SELECT LOWER(NAME) FROM Players;
```

## UPPER[](#upper)

### Description[](#description-14)

Converts a string to uppercase.

```
{UPPER | UCASE} (string)
```

### Parameters[](#parameters-14)

`string` - the string to convert

### Example[](#example-14)

Return the last name for each Player in uppercase:

```
SELECT UPPER(last_name) "LastNameUpperCase" FROM Players;
```

## LEFT[](#left)

### Description[](#description-15)

Returns the specified number of leftmost characters in a string.

```
LEFT(string, int)
```

### Parameters[](#parameters-15)

-   `string` - the string to extract the characters from
    
-   `int` - the number of characters to extract
    

### Example[](#example-15)

Extract the first 3 letters from Players' names:

```
SELECT LEFT(NAME, 3) FROM Players;
```

## RIGHT[](#right)

Returns the specified number of rightmost characters in a string.

```
RIGHT(string, int)
```

### Parameters[](#parameters-16)

-   `string` - the string to extract the characters from
    
-   `int` - the number of characters to extract
    

### Example[](#example-16)

Extract the first 3 letters from Players' names:

```
SELECT RIGHT(NAME, 3) FROM Players;
```

## LOCATE[](#locate)

### Description[](#description-16)

Returns the location of a search string in a string. If a start position is used, the characters before it are ignored. If the position is negative, the rightmost location is returned. Zero (0) is returned if the search string is not found.

```
LOCATE(searchString, string [, startInt])
```

### Parameters[](#parameters-17)

-   `searchString` - the string to search for
    
-   `string` - the string to search in
    
-   `startInt` - start position for the search
    

### Example[](#example-17)

Search for period (.) in Players' names:

```
SELECT LOCATE('.', NAME) FROM Players;
```

## POSITION[](#position)

### Description[](#description-17)

Returns the location of a search string in a string. A simplified version of [LOCATE](#locate).

```
POSITION(searchString, string)
```

### Parameters[](#parameters-18)

-   `searchString` - the string to search for
    
-   `string` - the string to search in
    

### Example[](#example-18)

Search for period (.) in Players' names:

```
SELECT POSITION('.', NAME) FROM Players;
```

## LPAD[](#lpad)

### Description[](#description-18)

Left-pads the string to the specified length. If the length is shorter than the string, the string is truncated at the end. If the padding string is not set, spaces are used.

```
LPAD(string, int[, paddingString])
```

### Parameters[](#parameters-19)

-   `string` - the string to pad
    
-   `int` - the length to pad to
    
-   `paddingString` - the string to pad with
    

### Example[](#example-19)

Left-pad Players' amount with asterisks to the length of 10:

```
SELECT LPAD(AMOUNT, 10, '*') FROM Players;
```

## RPAD[](#rpad)

### Description[](#description-19)

Right-pad the string to the specified length. If the length is shorter than the string, the string is truncated. If the padding string is not set, spaces are used.

```
RPAD(string, int[, paddingString])
```

### Parameters[](#parameters-20)

-   `string` - the string to pad
    
-   `int` - the length to pad to
    
-   `paddingString` - the string to pad with
    

### Example[](#example-20)

Right-pad Players' text with dashed to the length of 10:

```
SELECT RPAD(TEXT, 10, '-') FROM Players;
```

## LTRIM[](#ltrim)

### Description[](#description-20)

Removes all leading spaces from a string.

```
LTRIM(string)
```

### Parameters[](#parameters-21)

`string` - the string to left-trim

### Example[](#example-21)

Remove leading spaces from Players' names:

```
SELECT LTRIM(NAME) FROM Players;
```

## RTRIM[](#rtrim)

### Description[](#description-21)

Removes all trailing spaces from a string.

```
RTRIM(string)
```

### Parameters[](#parameters-22)

`string` - the string to right-trim

### Example[](#example-22)

Remove trailing spaces from Players' names:

```
SELECT RTRIM(NAME) FROM Players;
```

## TRIM[](#trim)

### Description[](#description-22)

Removes all leading spaces, trailing spaces, or spaces at both ends, from a string. Other characters can be removed as well.

```
TRIM ([{LEADING | TRAILING | BOTH} [string] FROM] string)
```

### Parameters[](#parameters-23)

-   `LEADING | TRAILING | BOTH` - indicates where to trim
    
-   `[string]` - (optional) what character(s)/string to trim
    
-   `string` - the string to trim
    

### Example[](#example-23)

Trim underscore characters on both ends of Players' names:

```
SELECT TRIM(BOTH '_' FROM NAME) FROM Players;
```

## REGEXP\_REPLACE[](#regexp_replace)

### Description[](#description-23)

Replaces each substring that matches a regular expression. For details, see the [Java String.replaceAll() method](https://www.javatpoint.com/java-string-replaceall). If any parameter is `NULL` (except the optional `flagsString` parameter), the result is `NULL`.

```
REGEXP_REPLACE(inputString, regexString, replacementString [, flagsString])
```

### Parameters[](#parameters-24)

-   `inputString` - the string to replace substrings in
    
-   `regexString` - the regular expression to match
    
-   `replacementString` - the string to replace the matching substrings with
    
-   `flagString` - possible values are:
    
    -   'i' enables case insensitive matching (Pattern.CASE\_INSENSITIVE)
        
    -   'c' disables case insensitive matching (Pattern.CASE\_INSENSITIVE)
        
    -   'n' allows the period to match the newline character (Pattern.DOTALL)
        
    -   'm' enables multiline mode (Pattern.MULTILINE)
        
        Other symbols cause an exception. Multiple symbols can be used in the `flagsString` parameter; for example, 'im'. Later flags override former ones; for example, 'ic' is equivalent to case sensitive, matching 'c'.
        
    

### Example[](#example-24)

Returns the names from the Players table with 'w' replaced by 'W':

```
SELECT REGEXP_REPLACE(name, 'w+', 'W', 'i') FROM Players;
```

## REGEXP\_LIKE[](#regexp_like)

### Description[](#description-24)

Matches string to a regular expression. For details, see the [Java Matcher.find() method](https://www.javatpoint.com/post/java-matcher-find-method). If any parameter is `NULL` (except the optional `flagsString` parameter), the result is `NULL`.

```
REGEXP_LIKE(inputString, regexString [, flagsString])
```

### Parameters[](#parameters-25)

-   `inputString` - the string to match regular expression
    
-   `regexString` - the regular expression to match
    
-   `flagString` - possible values are:
    
    -   'i' enables case insensitive matching (Pattern.CASE\_INSENSITIVE)
        
    -   'c' disables case insensitive matching (Pattern.CASE\_INSENSITIVE)
        
    -   'n' allows the period to match the newline character (Pattern.DOTALL)
        
    -   'm' enables multiline mode (Pattern.MULTILINE)
        
        Other symbols cause an exception. Multiple symbols can be used in the `flagsString` parameter; for example, 'im'. Later flags override former ones; for example, 'ic' is equivalent to case sensitive, matching 'c'.
        
    

### Example[](#example-25)

Returns Players whose names match 'A-M':

```
SELECT REGEXP_LIKE(name, '[A-M ]*', 'i') FROM Players;
```

## REPEAT[](#repeat)

### Description[](#description-25)

Returns a string repeated the specified number of times.

```
REPEAT(string, int)
```

### Parameters[](#parameters-26)

-   `string` - the string to repeat
    
-   `int` - the number of times to repeat the string
    

### Example[](#example-26)

Repeat Players' names 10 times:

```
SELECT REPEAT(NAME || ' ', 10) FROM Players;
```

## REPLACE[](#replace)

### Description[](#description-26)

Replaces all occurrences of a search string in the specified text with another string. If no replacement is specified, the search string is removed from the text. If any parameter is `NULL`, the result is `NULL`.

```
REPLACE(string, searchString [, replacementString])
```

### Parameters[](#parameters-27)

-   `string` - the text to replace the string in
    
-   `searchString` - the string to replace
    
-   `replacementString` - the string to replace with
    

### Example[](#example-27)

Remove spaces from Players' names:

```
SELECT REPLACE(NAME, ' ') FROM Players;
```

## SOUNDEX[](#soundex)

### Description[](#description-27)

Returns a four-character code representing the `SOUNDEX` of a string. For details, see [http://www.archives.gov/genealogy/census/soundex.html](https://www.archives.gov/genealogy/census/soundex.html). Returns a `string`.

```
SOUNDEX(string)
```

### Parameters[](#parameters-28)

`string` - the SOUNDEX source string

### Example[](#example-28)

Return the four-character code representing the `SOUNDEX` of Players' names:

```
SELECT SOUNDEX(NAME) FROM Players;
```

## SPACE[](#space)

### Description[](#description-28)

Returns a `string` consisting of the specified number of spaces.

```
SPACE(int)
```

### Parameters[](#parameters-29)

`int` - the number of spaces in the string to return

### Example[](#example-29)

Return two columns: Players' names in one, 80 spaces in the other:

```
SELECT name, SPACE(80) FROM Players;
```

## STRINGDECODE[](#stringdecode)

### Description[](#description-29)

Converts an encoded string using the Java string literal encoding format. Special characters are `\b`, `\t`, `\n`, `\f`, `\r`, `\"`, `\`, `\<octal>`, `\u<unicode>`. Returns a `string`.

```
STRINGDECODE(string)
```

### Parameters[](#parameters-30)

`string` - the string that contains the special characters

### Example[](#example-30)

Convert the line break symbol (\\n) to a format Java can work with:

```
STRINGDECODE('Lines 1\nLine 2');
```

## STRINGENCODE[](#stringencode)

### Description[](#description-30)

Encodes special characters in a string using the Java string literal encoding format. Special characters are `\b`, `\t`, `\n`, `\f`, `\r`, `\"`, `\`, `\<octal>`, `\u<unicode>`. Returns a `string`.

```
STRINGENCODE(string)
```

### Parameters[](#parameters-31)

`string` - the string that contains the special characters

### Example[](#example-31)

Encode the previously converted line break symbol (\\n):

```
STRINGENCODE(STRINGDECODE('Lines 1\nLine 2'))
```

## STRINGTOUTF8[](#stringtoutf8)

### Description[](#description-31)

Encodes a string to a byte array using the UTF8 encoding format. Returns `bytes`.

```
STRINGTOUTF8(string)
```

### Parameters[](#parameters-32)

`string` - the string to encode

### Example[](#example-32)

Encode Players' names:

```
SELECT STRINGTOUTF8(name) FROM Players;
```

## SUBSTRING[](#substring)

### Description[](#description-32)

Returns a substring of a string starting at the specified position. Also supported is: `SUBSTRING(string [FROM start] [FOR length])`.

```
{SUBSTRING | SUBSTR} (string, startInt [, lengthInt])
SUBSTRING(string [FROM start] [FOR length])
```

### Parameters[](#parameters-33)

-   `string` - the string to extract the substring from
    
-   `startInt` | `start` - the starting position for the substring to extract; if negative, the start index is relative to the end of the string
    
-   `lengthInt` | `length` - (optional) the length of the substring to extract
    

### Example[](#example-33)

From Players names, extract a 5-character long substring that starts at the second position:

```
SELECT SUBSTR(name, 2, 5) FROM Players;
```

## TO\_CHAR[](#to_char)

### Description[](#description-33)

Formats a timestamp, number, or text.

```
TO_CHAR(value [, formatString[, nlsParamString]])
```

### Parameters[](#parameters-34)

-   `formatString` - the string to be formatted
    
-   `nlsParamString` - the format to use
    

### Example:[](#example-34)

Format the '2010-01-01 00:00:00' timestamp to 'DD MON, YYYY':

```
TO_CHAR(TIMESTAMP '2010-01-01 00:00:00', 'DD MON, YYYY')
```

## TRANSLATE[](#translate)

### Description[](#description-34)

Replaces a sequence of characters in a string with another set of characters.

```
TRANSLATE(value , searchString, replacementString]])
```

### Parameters[](#parameters-35)

-   `value` - the string that contains the character sequence(s)
    
-   `searchString` - the sequence of characters to replace
    
-   `replacementString` - the string to replace the sequence of characters with
    

### Example[](#example-35)

Replace 'el' from 'Hello world!' with 'EL':

```
TRANSLATE('Hello world', 'el', 'EL')
```

## UTF8TOSTRING[](#utf8tostring)

### Description[](#description-35)

Decodes a byte array in the UTF8 format to a string.

```
UTF8TOSTRING(bytes)
```

### Parameters[](#parameters-36)

`bytes` - the array to decode

### Example[](#example-36)

Decode byte array derived from Players' names:

```
SELECT UTF8TOSTRING(name) FROM Players;
```

## XMLATTR[](#xmlattr)

### Description[](#description-36)

Creates an XML attribute element in the `name=value` form. The value is encoded as XML text. Returns a `string`.

```
XMLATTR(nameString, valueString)
```

### Parameters[](#parameters-37)

-   `nameString` - the 'name' part of the element
    
-   `valueString` - the 'value' part of the element
    

### Example[](#example-37)

Create the 'href=http://h2database.com' XML element:

```
XMLNODE('a', XMLATTR('href', 'http://h2database.com'))
```

## XMLNODE[](#xmlnode)

### Description[](#description-37)

Creates an XML node element. An empty or null attribute string means no attributes are set. An empty or null content string means the node is empty. The content is indented by default if it contains a newline. This function returns a `string`.

```
XMLNODE(elementString [, attributesString [, contentString [, indentBoolean]]])
```

### Parameters[](#parameters-38)

-   `elementString` - the name of the element/node to create
    
-   `attributeString` - the list of attributes for the element/node to create
    
-   `contentString` - the element/node content
    
-   `indentBoolean` - 'yes' to indent the node element, 'no' otherwise
    

### Example[](#example-38)

Create 'a' XML node element at the H2 indentation level:

```
XMLNODE('a', XMLATTR('href', 'http://h2database.com'), 'H2')
```

## XMLCOMMENT[](#xmlcomment)

### Description[](#description-38)

Creates an XML comment. Two dashes (`--`) are converted to `- -`. This function returns a `string`.

```
XMLCOMMENT(commentString)
```

### Parameters[](#parameters-39)

`commentString` - the comment content

### Example[](#example-39)

Create the 'Test' comment:

```
XMLCOMMENT('Test')
```

## XMLCDATA[](#xmlcdata)

### Description[](#description-39)

Creates an XML CDATA element. If the value contains `]]>`, an XML text element is created instead. This function returns a `string`.

```
XMLCDATA(valueString)
```

### Parameters[](#parameters-40)

`valueString` - the CDATA/text element’s content

### Example[](#example-40)

Create the 'data' CDATA element:

```
XMLCDATA('data')
```

## XMLSTARTDOC[](#xmlstartdoc)

### Description[](#description-40)

Returns the XML declaration. The result is always `<?xml version=1.0?>`.

```
XMLSTARTDOC()
```

### Example[](#example-41)

```
XMLSTARTDOC()
```

## XMLTEXT[](#xmltext)

### Description[](#description-41)

Creates an XML text element, returns a `string`.

```
XMLTEXT(valueString [, escapeNewlineBoolean])
```

### Parameters[](#parameters-41)

-   `valueString` - the text element’s content
    
-   `escapeNewlineBoolean` - if 'yes', newline and linefeed are converted to an XML entity (`&#`)
    

### Example[](#example-42)

Create the 'data' XML text element:

```
XMLTEXT(('data'))
```
# Date and Time Functions | GridGain Documentation

# Date and Time Functions[](#date-and-time-functions)

## CURRENT\_DATE[](#current_date)

### Description[](#description)

Returns the current date. When called multiple times within a transaction, returns the same value.

```
{CURRENT_DATE [()] | CURDATE() | SYSDATE | TODAY}
```

### Example[](#example)

Return the current date:

```
CURRENT_DATE()
```

## CURRENT\_TIME[](#current_time)

### Description[](#description-2)

Returns the current time.

```
{CURRENT_TIME [ () ] | CURTIME()}
```

### Example[](#example-2)

Return the current time:

```
CURRENT_TIME()
```

## CURRENT\_TIMESTAMP[](#current_timestamp)

### Description[](#description-3)

Returns the current timestamp. When called multiple times within a transaction, returns the same value.

```
{CURRENT_TIMESTAMP [([int])] | NOW([int])}
```

### Parameters[](#parameters)

`int` - (optional) the precision parameter for nanoseconds - the number of digits after the decimal point (for example, 9 is nanoseconds)

### Example[](#example-3)

Return the current timestamp:

```
CURRENT_TIMESTAMP()
```

## DATEADD | TIMESTAMPADD[](#dateadd-timestampadd)

### Description[](#description-4)

Adds units to the timestamp or date (if the value is positive). Subtracts units from the timestamp or date (if the value is negative). The DATEADD function returns a `timestamp`. The TIMESTAMPADD function returns a `long`.

```
{DATEADD | TIMESTAMPADD} (unitString, addIntLong, timestamp)
```

### Parameters[](#parameters-2)

-   `unitString` - the unit to add or subtract (see the [EXTRACT](#extract) function for supported units)
    
-   `addIntLong` - the number of units to add or subtract; may be `long` when manipulating milliseconds, `int` otherwise
    
-   `timestamp` - the timestamp to add units to
    

### Example[](#example-4)

Add 1 month to Jan 31 2021:

```
DATEADD('MONTH', 1, DATE '2001-01-31')
```

## DATEDIFF[](#datediff)

### Description[](#description-5)

Returns the number of specified units that differentiate between two timestamps as a `long`.

```
{DATEDIFF | TIMESTAMPDIFF} (unitString, aTimestamp, bTimestamp)
```

### Parameters[](#parameters-3)

-   `unitString` - the unit to consider (see the [EXTRACT](#extract) function for supported units)
    
-   `aTimestamp` - the first timestamp to compare
    
-   `bTimestamp` - the second timestamp to compare
    

### Example[](#example-5)

Return the number of years that differentiate T1.CREATED from T2.CREATED:

```
DATEDIFF('YEAR', T1.CREATED, T2.CREATED)
```

## DAYNAME[](#dayname)

### Description[](#description-6)

Returns the name of the day of the week (in English).

```
DAYNAME(date)
```

### Parameters[](#parameters-4)

`date` - the date to process

### Example[](#example-6)

Return the day of the week for CREATED:

```
DAYNAME(CREATED)
```

## DAY\_OF\_MONTH[](#day_of_month)

### Description[](#description-7)

Returns the day of the month (1-31).

```
DAY_OF_MONTH(date)
```

### Parameters[](#parameters-5)

`date` - the date to process

### Example[](#example-7)

Return the day of the month for CREATED:

```
DAY_OF_MONTH(CREATED)
```

## DAY\_OF\_WEEK[](#day_of_week)

### Description[](#description-8)

Returns the day of the week (1-7, 1=Sunday).

```
DAY_OF_WEEK(date)
```

### Parameters[](#parameters-6)

`date` - the date to process

### Example[](#example-8)

Return the number of the day of the week for CREATED:

```
DAY_OF_WEEK(CREATED)
```

## DAY\_OF\_YEAR[](#day_of_year)

### Description[](#description-9)

Returns the day of the year (1-366).

```
DAY_OF_YEAR(date)
```

### Parameters[](#parameters-7)

`date` - the date to process

### Example[](#example-9)

Return the number of the day of the year for CREATED:

```
DAY_OF_YEAR(CREATED)
```

## EXTRACT[](#extract)

### Description[](#description-10)

Returns a specific value from a timestamp as an `int`.

```
EXTRACT ({EPOCH | YEAR | YY | QUARTER | MONTH | MM | WEEK | ISO_WEEK
| DAY | DD | DAY_OF_YEAR | DOY | DAY_OF_WEEK | DOW | ISO_DAY_OF_WEEK
| HOUR | HH | MINUTE | MI | SECOND | SS | MILLISECOND | MS
| MICROSECOND | MCS | NANOSECOND | NS}
FROM timestamp)
```

### Parameters[](#parameters-8)

`timestamp` - the timestamp to process

### Example[](#example-10)

Extract the value of the 'seconds' part from CURRENT\_TIMESTAMP:

```
EXTRACT(SECOND FROM CURRENT_TIMESTAMP)
```

## FORMATDATETIME[](#formatdatetime)

### Description[](#description-11)

Formats a date, time, or timestamp as a `string`. The main format characters are: `y` year, `M` month, `d` day, `H` hour, `m` minute, and `s` second. For more details, see the [java.text.SimpleDateFormat method](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/text/SimpleDateFormat.html).

```
FORMATDATETIME (timestamp, formatString [,localeString [,timeZoneString]])
```

### Parameters[](#parameters-9)

-   `timestamp` - the timestamp to process
    
-   `formatString` - the format to apply
    
-   `localeString` - the name of the locale
    
-   `timeZoneString` - the time zone
    

### Example[](#example-11)

Format 2001-02-03 04:05:06 in English for GMT using the 'EEE, d MMM yyyy HH:mm:ss z' format:

```
FORMATDATETIME(TIMESTAMP '2001-02-03 04:05:06', 'EEE, d MMM yyyy HH:mm:ss z', 'en', 'GMT')
```

## HOUR[](#hour)

### Description[](#description-12)

Returns the hour (0-23) from a timestamp.

```
HOUR(timestamp)
```

### Parameters[](#parameters-10)

`timestamp` - the timestamp to process

### Example[](#example-12)

Return the number of hours for CREATED:

```
HOUR(CREATED)
```

## MINUTE[](#minute)

### Description[](#description-13)

Returns the minutes (0-59) from a timestamp.

```
MINUTE(timestamp)
```

### Parameters[](#parameters-11)

`timestamp` - the timestamp to process

### Example[](#example-13)

Return the number of minutes from CREATED:

```
MINUTE(CREATED)
```

## MONTH[](#month)

### Description[](#description-14)

Returns the month (1-12) from a timestamp.

```
MONTH(timestamp)
```

### Parameters[](#parameters-12)

`` `timestamp `` - the timestamp to process

### Example[](#example-14)

Return the months from CREATED:

```
MONTH(CREATED)
```

## MONTHNAME[](#monthname)

### Description[](#description-15)

Returns the name of the month (in English).

```
MONTHNAME(date)
```

### Parameters[](#parameters-13)

`date` - the date to process

### Example[](#example-15)

Return the name of the month from CREATED:

```
MONTHNAME(CREATED)
```

## PARSEDATETIME[](#parsedatetime)

### Description[](#description-16)

Parses a string and converts it to `timestamp`. The main characters in the string format are: `y` year, `M` month, `d` day, `H` hour, `m` minute, and `s` second. For more details, see the [java.text.SimpleDateFormat method](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/text/SimpleDateFormat.html).

```
PARSEDATETIME(string, formatString [, localeString [, timeZoneString]])
```

### Parameters[](#parameters-14)

-   `string` - the string top parse
    
-   `formatString` - the string format
    
-   `localeString` - the name of the locale
    
-   `timeZoneString` - the time zone
    

### Example[](#example-16)

Convert to `timestamp` the 'Sat, 3 Feb 2001 03:05:06 GMT':

```
PARSEDATETIME('Sat, 3 Feb 2001 03:05:06 GMT', 'EEE, d MMM yyyy HH:mm:ss z', 'en', 'GMT')
```

## QUARTER[](#quarter)

### Description[](#description-17)

Returns the quarter (1-4) from a timestamp.

```
QUARTER(timestamp)
```

### Parameters[](#parameters-15)

`timestamp` - the timestamp to process

### Example[](#example-17)

Return the quarter for CREATED:

```
QUARTER(CREATED)
```

## SECOND[](#second)

### Description[](#description-18)

Returns the second (0-59) from a timestamp.

```
SECOND(timestamp)
```

### Parameters[](#parameters-16)

`timestamp` - the timestamp to process

### Example[](#example-18)

Return the number of seconds for CREATED:

```
SECOND(CREATED)
```

## WEEK[](#week)

### Description[](#description-19)

Returns the week (1-53) from a timestamp using the current system locale.

```
WEEK(timestamp)
```

### Parameters[](#parameters-17)

`timestamp` - the timestamp to process

### Example[](#example-19)

Return the number of the week from CREATED:

```
WEEK(CREATED)
```

## YEAR[](#year)

### Description[](#description-20)

Returns the year from a timestamp.

```
YEAR(timestamp)
```

### Parameters[](#parameters-18)

`timestamp` - the timestamp to process

### Example[](#example-20)

Return the year from CREATED:

```
YEAR(CREATED)
```

# System Functions | GridGain Documentation

# System Functions[](#system-functions)

## CASEWHEN[](#casewhen)

### Description[](#description)

Returns 'aValue' if the Boolean expression is true, 'bValue' otherwise.

```
CASEWHEN (boolean , aValue , bValue)
```

### Parameters[](#parameters)

-   `boolean` - the value to evaluate
    
-   `aValue` - the value to return if Boolean is `true`
    
-   `bValue` - value to return if Boolean is `false`
    

### Example[](#example)

Evaluate the ID and return the corresponding value:

```
CASEWHEN(ID=1, 'A', 'B')
```

## CAST[](#cast)

### Description[](#description-2)

Converts a value to another data type using one the following conversion rules:

-   When converting a number to a Boolean, ``0 is considered `false`` and any other value is considered `true`
    
-   When converting a Boolean to a number, `false` is `0` and `true` is `1`
    
-   When converting a number to a number of another type, the value is checked for overflow
    
-   When converting a number to a binary type, the number of bytes matches the precision
    
-   When converting a string to a binary type, it is hex-encoded
    
-   A hex string can be converted to the binary type and then to a number; if a direct conversion is not possible, the value is first converted to a string
    

```
CAST (value AS dataType)
```

### Parameters[](#parameters-2)

-   `value` - the value to convert
    
-   `dataType` - the data type to convert the value to
    

### Examples[](#examples)

```
CAST(NAME AS INT);
CAST(65535 AS BINARY);
CAST(CAST('FFFF' AS BINARY) AS INT);
```

## COALESCE | NVL[](#coalesce-nvl)

### Description[](#description-3)

Returns the first value that is not `NULL`.

```
{COALESCE | NVL } (aValue, bValue [,...])
```

### Parameters[](#parameters-3)

`aValue`, `bValue` - values to process

### Example[](#example-2)

Return the first non-null value out of A, B, and C:

```
COALESCE(A, B, C)
```

## CONVERT[](#convert)

### Description[](#description-4)

Converts a value to another data type.

```
CONVERT (value , dataType)
```

### Parameters[](#parameters-4)

-   `value` - the value to convert
    
-   `dataType` - the data type to convert the value to
    

### Example[](#example-3)

Convert NAME to an integer:

```
CONVERT(NAME, INT)
```

## DECODE[](#decode)

### Description[](#description-5)

Returns the first matching value. `NULL` matches `` NULL` ``. If no match is found, `NULL` or the last parameter is returned (if the parameter count is even).

```
DECODE(value, whenValue, thenValue [,...])
```

### Parameters[](#parameters-5)

-   `value` - the value to match against
    
-   `whenValue`, `thenValue`, etc. - values to check
    

### Example[](#example-4)

Return the first value that matches `random number greater than 0.5`:

```
DECODE(RAND()>0.5, 0, 'Red', 1, 'Black')
```

## GREATEST[](#greatest)

### Description[](#description-6)

Returns the greatest value that is not `NULL`, or `NULL` if all values are `NULL`.

```
GREATEST(aValue, bValue [,...])
```

### Parameters[](#parameters-6)

`aValue`, `bValue` - values to process

### Example[](#example-5)

Return the greatest values out of 1, 2, and 3:

```
GREATEST(1, 2, 3)
```

## IFNULL[](#ifnull)

### Description[](#description-7)

Returns 'a' if the value is not `NULL`, 'b' otherwise.

```
IFNULL(aValue, bValue)
```

### Parameters[](#parameters-7)

`aValue`, `bValue` - values to process

### Example[](#example-6)

Check NULL and space ('') for being `NULL`:

```
IFNULL(NULL, '')
```

## LEAST[](#least)

### Description[](#description-8)

Returns the smallest value that is not NULL, or NULL if all values are NULL.

```
LEAST(aValue, bValue [,...])
```

### Parameters[](#parameters-8)

### Example[](#example-7)

```
LEAST(1, 2, 3)
```

## NULLIF[](#nullif)

### Description[](#description-9)

Returns NULL if 'a' equals 'b', otherwise returns 'a'.

```
NULLIF(aValue, bValue)
```

### Parameters[](#parameters-9)

`aValue`, `bValue` - values to process

### Example[](#example-8)

Check if A=B:

```
NULLIF(A, B)
```

## NVL2[](#nvl2)

### Description[](#description-10)

If the test value is `NULL`, returns 'bValue'. Otherwise, returns 'aValue'. The data type of the returned value is the same as that of 'aValue'.

```
NVL2(testValue, aValue, bValue)
```

### Parameters[](#parameters-10)

-   `testValue` - the value to test for `NULL`
    
-   `aValue` - the value to return if testVale is not `NULL`
    
-   `bValue` - the value to return if testVale is `NULL`
    

### Example[](#example-9)

```
Test if X is `NULL`:
```

NVL2(X, 'not null', 'null')

## TABLE | TABLE\_DISTINCT[](#table-table_distinct)

### Description[](#description-11)

Returns the result set that matches the criteria. TABLE\_DISTINCT removes duplicate rows.

```
TABLE | TABLE_DISTINCT	(name dataType = expression)
```

### Parameters[](#parameters-11)

-   `name` - the table name
    
-   `dataType` - the data type to return
    
-   `expression` - teh expression that defines the dataType
    

### Example[](#example-10)

Return everything from the table:

```
SELECT * FROM TABLE(ID INT=(1, 2), NAME VARCHAR=('Hello', 'World'))
```
# Data Types | GridGain Documentation

# Data Types[](#data-types)

The page contains a list of SQL data types available in GridGain such as string, numeric, and date/time types.

Every SQL type is mapped to a programming language or driver specific type that is supported by GridGain natively.

## BOOLEAN[](#boolean)

Possible values: TRUE and FALSE.

Mapped to:

-   Java/JDBC: `java.lang.Boolean`
    
-   .NET/C#: `bool`
    
-   C/C++: `bool`
    
-   ODBC: `SQL_BIT`
    

## BIGINT[](#bigint)

Possible values: \[`-9223372036854775808`, `9223372036854775807`\].

Mapped to:

-   Java/JDBC: `java.lang.Long`
    
-   .NET/C#: `long`
    
-   C/C++: `int64_t`
    
-   ODBC: `SQL_BIGINT`
    

## DECIMAL[](#decimal)

Possible values: Data type with fixed precision and scale.

Mapped to:

-   Java/JDBC: `java.math.BigDecimal`
    
-   .NET/C#: `decimal`
    
-   C/C++: `ignite::Decimal`
    
-   ODBC: `SQL_DECIMAL`
    

## DOUBLE[](#double)

Possible values: A floating point number.

Mapped to:

-   Java/JDBC: `java.lang.Double`
    
-   .NET/C#: `double`
    
-   C/C++: `double`
    
-   ODBC: `SQL_DOUBLE`
    

## INT[](#int)

Possible values: \[`-2147483648`, `2147483647`\].

Mapped to:

-   Java/JDBC: `java.lang.Integer`
    
-   .NET/C#: `int`
    
-   C/C++: `int32_t`
    
-   ODBC: `SQL_INTEGER`
    

## REAL[](#real)

Possible values: A single precision floating point number.

Mapped to:

-   Java/JDBC: `java.lang.Float`
    
-   .NET/C#: `float`
    
-   C/C++: `float`
    
-   ODBC: `SQL_FLOAT`
    

## SMALLINT[](#smallint)

Possible values: \[`-32768`, `32767`\].

Mapped to:

-   Java/JDBC: `java.lang.Short`
    
-   .NET/C#: `short`
    
-   C/C++: `int16_t`
    
-   ODBC: `SQL_SMALLINT`
    

## TINYINT[](#tinyint)

Possible values: \[`-128`, `127`\].

Mapped to:

-   Java/JDBC: `java.lang.Byte`
    
-   .NET/C#: `sbyte`
    
-   C/C++: `int8_t`
    
-   ODBC: `SQL_TINYINT`
    

## CHAR[](#char)

Possible values: A unicode String. This type is supported for compatibility with other databases and older applications.

Mapped to:

-   Java/JDBC: `java.lang.String`
    
-   .NET/C#: `string`
    
-   C/C++: `std::string`
    
-   ODBC: `SQL_CHAR`
    

## VARCHAR[](#varchar)

Possible values: A Unicode String.

Mapped to:

-   Java/JDBC: `java.lang.String`
    
-   .NET/C#: `string`
    
-   C/C++: `std::string`
    
-   ODBC: `SQL_VARCHAR`
    

## DATE[](#date)

Possible values: The date data type. The format is `yyyy-MM-dd`.

Mapped to:

Java/JDBC: `java.sql.Date` - .NET/C#: N/A - C/C++: `ignite::Date` - ODBC: `SQL_TYPE_DATE`

## TIME[](#time)

Possible values: The time data type. The format is `hh:mm:ss`.

Mapped to:

-   Java/JDBC: `java.sql.Time`
    
-   .NET/C#: N/A
    
-   C/C++: `ignite::Time`
    
-   ODBC: `SQL_TYPE_TIME`
    

## TIMESTAMP[](#timestamp)

Possible values: The timestamp data type. The format is `yyyy-MM-dd hh:mm:ss[.nnnnnnnnn]`.

Mapped to:

-   Java/JDBC: `java.sql.Timestamp, java.util.Date`
    
-   .NET/C#: `System.DateTime`
    
-   C/C++: `ignite::Timestamp`
    
-   ODBC: `SQL_TYPE_TIMESTAMP`
    

## BINARY[](#binary)

Possible values: Represents a byte array.

Mapped to:

-   Java/JDBC: `byte[]`
    
-   .NET/C#: `byte[]`
    
-   C/C++: `int8_t[]`
    
-   ODBC: `SQL_BINARY`
    

## GEOMETRY[](#geometry)

Possible values: A spatial geometry type, based on the `com.vividsolutions.jts` library. Normally represented in a textual format using the WKT (well-known text) format.

Mapped to:

-   Java/JDBC: Types from the `com.vividsolutions.jts` package.
    
-   .NET/C#: N/A
    
-   C/C++: N/A
    
-   ODBC: N/A
    

## UUID[](#uuid)

Possible values: Universally unique identifier. This is a 128 bit value.

Mapped to:

-   Java/JDBC: `java.util.UUID`
    
-   .NET/C#: `System.Guid`
    
-   C/C++: `ignite::Guid`
    
-   ODBC: `SQL_GUID`
