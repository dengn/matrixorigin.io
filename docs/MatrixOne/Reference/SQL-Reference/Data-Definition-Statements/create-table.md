# **CREATE TABLE**

## **Description**

Create a new table.

## **Syntax**

```
> CREATE [TEMPORARY] TABLE [IF NOT EXISTS] [db.]table_name [comment = "comment of table"];
(
    name1 type1 [comment 'comment of column'] [AUTO_INCREMENT] [[PRIMARY] KEY],
    name2 type2 [comment 'comment of column'],
    ...
)
    [partition_options]
```

### Explanations

#### Temporary Tables

You can use the `TEMPORARY` keyword when creating a table. A `TEMPORARY` table is visible only within the current session, and is dropped automatically when the session is closed. For more information, see [Temporary Tables](create-temporary-tables.md).

#### COMMENT

A comment for a column or a table can be specified with the `COMMENT` option.

- Up to 1024 characters long. The comment is displayed by the `SHOW CREATE TABLE` and `SHOW FULL COLUMNS` statements. It is also shown in the `COLUMN_COMMENT` column of the `INFORMATION_SCHEMA.COLUMNS` table.

#### AUTO_INCREMENT

The initial `AUTO_INCREMENT` value for the table.

An integer column can have the additional attribute `AUTO_INCREMENT`. When you insert a value of NULL (recommended) or 0 into an indexed AUTO_INCREMENT column, the column is set to the next sequence value. Typically this is value+1, where value is the largest value for the column currently in the table. AUTO_INCREMENT sequences begin with 1.

There can be only one `AUTO_INCREMENT` column per table, it must be indexed, and it cannot have a DEFAULT value. An `AUTO_INCREMENT` column works properly only if it contains only positive values. Inserting a negative number is regarded as inserting a very large positive number. This is done to avoid precision problems when numbers “wrap” over from positive to negative and also to ensure that you do not accidentally get an `AUTO_INCREMENT` column that contains 0.

#### Table PARTITION and PARTITIONS

```
partition_options:
 PARTITION BY
     { [LINEAR] HASH(expr)
     | [LINEAR] KEY [ALGORITHM={1 | 2}] (column_list)
     | RANGE{(expr) | COLUMNS(column_list)}
     | LIST{(expr) | COLUMNS(column_list)} }
 [PARTITIONS num]
 [SUBPARTITION BY
     { [LINEAR] HASH(expr)
     | [LINEAR] KEY [ALGORITHM={1 | 2}] (column_list) }
 ]
 [(partition_definition [, partition_definition] ...)]

partition_definition:
 PARTITION partition_name
     [VALUES
         {LESS THAN {(expr | value_list) | MAXVALUE}
         |
         IN (value_list)}]
     [[STORAGE] ENGINE [=] engine_name]
     [COMMENT [=] 'string' ]
     [DATA DIRECTORY [=] 'data_dir']
     [INDEX DIRECTORY [=] 'index_dir']
     [MAX_ROWS [=] max_number_of_rows]
     [MIN_ROWS [=] min_number_of_rows]
     [TABLESPACE [=] tablespace_name]
```

Partitions can be modified, merged, added to tables, and dropped from tables.

- **PARTITION BY**

If used, a partition_options clause begins with PARTITION BY. This clause contains the function that is used to determine the partition; the function returns an integer value ranging from 1 to num, where num is the number of partitions.

- **HASH(expr)**

Hashes one or more columns to create a key for placing and locating rows. expr is an expression using one or more table columns. For example, these are both valid CREATE TABLE statements using PARTITION BY HASH:

```
CREATE TABLE t1 (col1 INT, col2 CHAR(5))
    PARTITION BY HASH(col1);

CREATE TABLE t1 (col1 INT, col2 CHAR(5), col3 DATETIME)
    PARTITION BY HASH ( YEAR(col3) );
```

- **KEY(column_list)**

This is similar to `HASH`. The column_list argument is simply a list of 1 or more table columns (maximum: 16). This example shows a simple table partitioned by key, with 4 partitions:

```
CREATE TABLE tk (col1 INT, col2 CHAR(5), col3 DATE)
    PARTITION BY KEY(col3)
    PARTITIONS 4;
```

For tables that are partitioned by key, you can employ linear partitioning by using the `LINEAR` keyword. This has the same effect as with tables that are partitioned by `HASH`. This example uses linear partitioning by `key` to distribute data between 5 partitions:

```
CREATE TABLE tk (col1 INT, col2 CHAR(5), col3 DATE)
    PARTITION BY LINEAR KEY(col3)
    PARTITIONS 5;
```

- **RANGE(expr)**

In this case, expr shows a range of values using a set of `VALUES LESS THAN` operators. When using range partitioning, you must define at least one partition using `VALUES LESS THAN`. You cannot use `VALUES IN` with range partitioning.

`PARTITION ... VALUES LESS THAN ...` statements work in a consecutive fashion. `VALUES LESS THAN MAXVALUE` works to specify “leftover” values that are greater than the maximum value otherwise specified.

The clauses must be arranged in such a way that the upper limit specified in each successive `VALUES LESS THAN` is greater than that of the previous one, with the one referencing `MAXVALUE` coming last of all in the list.

- **RANGE COLUMNS(column_list)**

This variant on `RANGE` facilitates partition pruning for queries using range conditions on multiple columns (that is, having conditions such as `WHERE a = 1 AND b < 10 or WHERE a = 1 AND b = 10 AND c < 10)`. It enables you to specify value ranges in multiple columns by using a list of columns in the COLUMNS clause and a set of column values in each `PARTITION ... VALUES LESS THAN (value_list)` partition definition clause. (In the simplest case, this set consists of a single column.) The maximum number of columns that can be referenced in the column_list and value_list is 16.

The column_list used in the `COLUMNS` clause may contain only names of columns; each column in the list must be one of the following MySQL data types: the integer types; the string types; and time or date column types. Columns using BLOB, TEXT, SET, ENUM, BIT, or spatial data types are not permitted; columns that use floating-point number types are also not permitted. You also may not use functions or arithmetic expressions in the `COLUMNS` clause.

The `VALUES LESS THAN` clause used in a partition definition must specify a literal value for each column that appears in the COLUMNS() clause; that is, the list of values used for each `VALUES LESS THAN` clause must contain the same number of values as there are columns listed in the `COLUMNS` clause. An attempt to use more or fewer values in a `VALUES LESS THAN` clause than there are in the COLUMNS clause causes the statement to fail with the error Inconsistency in usage of column lists for partitioning.... You cannot use `NULL` for any value appearing in `VALUES LESS THAN`. It is possible to use MAXVALUE more than once for a given column other than the first, as shown in this example:

```
CREATE TABLE rc (
    a INT NOT NULL,
    b INT NOT NULL
)
PARTITION BY RANGE COLUMNS(a,b) (
    PARTITION p0 VALUES LESS THAN (10,5),
    PARTITION p1 VALUES LESS THAN (20,10),
    PARTITION p2 VALUES LESS THAN (50,MAXVALUE),
    PARTITION p3 VALUES LESS THAN (65,MAXVALUE),
    PARTITION p4 VALUES LESS THAN (MAXVALUE,MAXVALUE)
);
```

Each value used in a `VALUES LESS THAN` value list must match the type of the corresponding column exactly; no conversion is made. For example, you cannot use the string '1' for a value that matches a column that uses an integer type (you must use the numeral 1 instead), nor can you use the numeral 1 for a value that matches a column that uses a string type (in such a case, you must use a quoted string: '1').

- **LIST(expr)**

This is useful when assigning partitions based on a table column with a restricted set of possible values, such as a state or country code. In such a case, all rows pertaining to a certain state or country can be assigned to a single partition, or a partition can be reserved for a certain set of states or countries. It is similar to RANGE, except that only VALUES IN may be used to specify permissible values for each partition.

`VALUES IN` is used with a list of values to be matched. For instance, you could create a partitioning scheme such as the following:

```
CREATE TABLE client_firms (
    id   INT,
    name VARCHAR(35)
)
PARTITION BY LIST (id) (
    PARTITION r0 VALUES IN (1, 5, 9, 13, 17, 21),
    PARTITION r1 VALUES IN (2, 6, 10, 14, 18, 22),
    PARTITION r2 VALUES IN (3, 7, 11, 15, 19, 23),
    PARTITION r3 VALUES IN (4, 8, 12, 16, 20, 24)
);
```

When using list partitioning, you must define at least one partition using VALUES IN. You cannot use VALUES LESS THAN with PARTITION BY LIST.

!!! note
     For tables partitioned by LIST, the value list used with VALUES IN must consist of integer values only. In MySQL 8.0, you can overcome this limitation using partitioning by LIST COLUMNS, which is described later in this section.

- **LIST COLUMNS(column_list)**

This variant on `LIST` facilitates partition pruning for queries using comparison conditions on multiple columns (that is, having conditions such as `WHERE a = 5 AND b = 5 or WHERE a = 1 AND b = 10 AND c = 5`). It enables you to specify values in multiple columns by using a list of columns in the COLUMNS clause and a set of column values in each `PARTITION ... VALUES IN` (value_list) partition definition clause.

The rules governing regarding data types for the column list used in `LIST COLUMNS(column_list)` and the value list used in `VALUES IN(value_list)` are the same as those for the column list used in `RANGE COLUMNS(column_list)` and the value list used in `VALUES LESS THAN(value_list)`, respectively, except that in the `VALUES IN` clause, `MAXVALUE` is not permitted, and you may use `NULL`.

There is one important difference between the list of values used for VALUES IN with `PARTITION BY LIST COLUMNS` as opposed to when it is used with `PARTITION BY LIST`. When used with `PARTITION BY LIST COLUMNS`, each element in the `VALUES IN` clause must be a set of column values; the number of values in each set must be the same as the number of columns used in the `COLUMNS` clause, and the data types of these values must match those of the columns (and occur in the same order). In the simplest case, the set consists of a single column. The maximum number of columns that can be used in the column_list and in the elements making up the value_list is 16.

The table defined by the following `CREATE TABLE` statement provides an example of a table using `LIST COLUMNS` partitioning:

```
CREATE TABLE lc (
    a INT NULL,
    b INT NULL
)
PARTITION BY LIST COLUMNS(a,b) (
    PARTITION p0 VALUES IN( (0,0), (NULL,NULL) ),
    PARTITION p1 VALUES IN( (0,1), (0,2), (0,3), (1,1), (1,2) ),
    PARTITION p2 VALUES IN( (1,0), (2,0), (2,1), (3,0), (3,1) ),
    PARTITION p3 VALUES IN( (1,3), (2,2), (2,3), (3,2), (3,3) )
);
```

- **PARTITIONS num**

The number of partitions may optionally be specified with a PARTITIONS num clause, where num is the number of partitions. If both this clause and any PARTITION clauses are used, num must be equal to the total number of any partitions that are declared using PARTITION clauses.

#### create_table_statement

![Create Table Diagram](https://github.com/matrixorigin/artwork/blob/main/docs/reference/create_table_statement.png?raw=true)

## **Examples**

- Example 1

```sql
> CREATE TABLE test(a int, b varchar(10));

> INSERT INTO test values(123, 'abc');

> SELECT * FROM test;
+------+---------+
|   a  |    b    |
+------+---------+
|  123 |   abc   |
+------+---------+
```

- Example 2

```sql
> create table t2 (a int, b int) comment = "事实表";
> show create table t2;
+-------+---------------------------------------------------------------------------------------+
| Table | Create Table                                                                          |
+-------+---------------------------------------------------------------------------------------+
| t2    | CREATE TABLE `t2` (
`a` INT DEFAULT NULL,
`b` INT DEFAULT NULL
) COMMENT='事实表',    |
+-------+---------------------------------------------------------------------------------------+
```

- Example 3

```sql
> create table t3 (a int comment '列的注释', b int) comment = "table";
> show create table t3;
+-------+----------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                             |
+-------+----------------------------------------------------------------------------------------------------------+
| t3    | CREATE TABLE `t3` (
`a` INT DEFAULT NULL COMMENT '列的注释',
`b` INT DEFAULT NULL
) COMMENT='table',     |
+-------+----------------------------------------------------------------------------------------------------------+
```

- Example 4

```sql
> CREATE TABLE tp1 (col1 INT, col2 CHAR(5), col3 DATE) PARTITION BY KEY(col3) PARTITIONS 4;
> show create table tp1;
+-------+-------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                          |
+-------+-------------------------------------------------------------------------------------------------------+
| tp1   | CREATE TABLE `tp1` (
`col1` INT DEFAULT NULL,
`col2` CHAR(5) DEFAULT NULL,
`col3` DATE DEFAULT NULL
) |
+-------+-------------------------------------------------------------------------------------------------------+
1 row in set (0.01 sec)

> CREATE TABLE tp2 (col1 INT, col2 CHAR(5), col3 DATE) PARTITION BY KEY(col3);
> show create table tp2;
+-------+-------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                          |
+-------+-------------------------------------------------------------------------------------------------------+
| tp2   | CREATE TABLE `tp2` (
`col1` INT DEFAULT NULL,
`col2` CHAR(5) DEFAULT NULL,
`col3` DATE DEFAULT NULL
) |
+-------+-------------------------------------------------------------------------------------------------------+
1 row in set (0.01 sec)

> CREATE TABLE tp4 (col1 INT, col2 CHAR(5), col3 DATE) PARTITION BY KEY ALGORITHM = 1 (col3);
> show create table tp4;
+-------+-------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                          |
+-------+-------------------------------------------------------------------------------------------------------+
| tp4   | CREATE TABLE `tp4` (
`col1` INT DEFAULT NULL,
`col2` CHAR(5) DEFAULT NULL,
`col3` DATE DEFAULT NULL
) |
+-------+-------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

> CREATE TABLE tp5 (col1 INT, col2 CHAR(5), col3 DATE) PARTITION BY LINEAR KEY ALGORITHM = 1 (col3) PARTITIONS 5;
> show create table tp5;
+-------+-------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                          |
+-------+-------------------------------------------------------------------------------------------------------+
| tp5   | CREATE TABLE `tp5` (
`col1` INT DEFAULT NULL,
`col2` CHAR(5) DEFAULT NULL,
`col3` DATE DEFAULT NULL
) |
+-------+-------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

> CREATE TABLE tp6 (col1 INT, col2 CHAR(5), col3 DATE) PARTITION BY KEY(col1, col2) PARTITIONS 4;
> show create table tp6;
+-------+-------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                          |
+-------+-------------------------------------------------------------------------------------------------------+
| tp6   | CREATE TABLE `tp6` (
`col1` INT DEFAULT NULL,
`col2` CHAR(5) DEFAULT NULL,
`col3` DATE DEFAULT NULL
) |
+-------+-------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

> CREATE TABLE tp7 (col1 INT NOT NULL PRIMARY KEY, col2 DATE NOT NULL, col3 INT NOT NULL, col4 INT NOT NULL) PARTITION BY KEY(col1) PARTITIONS 4;
> show create table tp7;
+-------+----------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                     |
+-------+----------------------------------------------------------------------------------------------------------------------------------+
| tp7   | CREATE TABLE `tp7` (
`col1` INT NOT NULL,
`col2` DATE NOT NULL,
`col3` INT NOT NULL,
`col4` INT NOT NULL,
PRIMARY KEY (`col1`)
) |
+-------+----------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

> CREATE TABLE tp8 (col1 INT, col2 CHAR(5)) PARTITION BY HASH(col1);
> show create table tp8;
+-------+-----------------------------------------------------------------------------+
| Table | Create Table                                                                |
+-------+-----------------------------------------------------------------------------+
| tp8   | CREATE TABLE `tp8` (
`col1` INT DEFAULT NULL,
`col2` CHAR(5) DEFAULT NULL
) |
+-------+-----------------------------------------------------------------------------+
1 row in set (0.00 sec)

> CREATE TABLE tp9 (col1 INT, col2 CHAR(5)) PARTITION BY HASH(col1) PARTITIONS 4;
> show create table tp9;
+-------+-----------------------------------------------------------------------------+
| Table | Create Table                                                                |
+-------+-----------------------------------------------------------------------------+
| tp9   | CREATE TABLE `tp9` (
`col1` INT DEFAULT NULL,
`col2` CHAR(5) DEFAULT NULL
) |
+-------+-----------------------------------------------------------------------------+
1 row in set (0.00 sec)

> CREATE TABLE tp10 (col1 INT, col2 CHAR(5), col3 DATETIME) PARTITION BY HASH (YEAR(col3));
> show create table tp10;
+-------+------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                               |
+-------+------------------------------------------------------------------------------------------------------------+
| tp10  | CREATE TABLE `tp10` (
`col1` INT DEFAULT NULL,
`col2` CHAR(5) DEFAULT NULL,
`col3` DATETIME DEFAULT NULL
) |
+-------+------------------------------------------------------------------------------------------------------------+
1 row in set (0.01 sec)

> CREATE TABLE tp11 (col1 INT, col2 CHAR(5), col3 DATE) PARTITION BY LINEAR HASH( YEAR(col3)) PARTITIONS 6;
> show create table tp11;
+-------+--------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                           |
+-------+--------------------------------------------------------------------------------------------------------+
| tp11  | CREATE TABLE `tp11` (
`col1` INT DEFAULT NULL,
`col2` CHAR(5) DEFAULT NULL,
`col3` DATE DEFAULT NULL
) |
+-------+--------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

> CREATE TABLE tp12 (col1 INT NOT NULL PRIMARY KEY, col2 DATE NOT NULL, col3 INT NOT NULL, col4 INT NOT NULL) PARTITION BY HASH(col1) PARTITIONS 4;
> show create table tp12;
+-------+-----------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                      |
+-------+-----------------------------------------------------------------------------------------------------------------------------------+
| tp12  | CREATE TABLE `tp12` (
`col1` INT NOT NULL,
`col2` DATE NOT NULL,
`col3` INT NOT NULL,
`col4` INT NOT NULL,
PRIMARY KEY (`col1`)
) |
+-------+-----------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.01 sec)

> CREATE TABLE tp13 (id INT NOT NULL PRIMARY KEY, fname VARCHAR(30), lname VARCHAR(30), hired DATE NOT NULL DEFAULT '1970-01-01', separated DATE NOT NULL DEFAULT '9999-12-31', job_code INT NOT NULL, store_id INT NOT NULL) PARTITION BY RANGE (id) (PARTITION p0 VALUES LESS THAN (6), PARTITION p1 VALUES LESS THAN (11), PARTITION p2 VALUES LESS THAN (16), PARTITION p3 VALUES LESS THAN (21));
> show create table tp13;
+-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                          |
+-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| tp13  | CREATE TABLE `tp13` (
`id` INT NOT NULL,
`fname` VARCHAR(30) DEFAULT NULL,
`lname` VARCHAR(30) DEFAULT NULL,
`hired` DATE NOT NULL,
`separated` DATE NOT NULL,
`job_code` INT NOT NULL,
`store_id` INT NOT NULL,
PRIMARY KEY (`id`)
) |
+-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

> CREATE TABLE tp14 (id INT NOT NULL, fname VARCHAR(30), lname VARCHAR(30), hired DATE NOT NULL DEFAULT '1970-01-01', separated DATE NOT NULL DEFAULT '9999-12-31', job_code INT, store_id INT) PARTITION BY RANGE ( YEAR(separated) ) ( PARTITION p0 VALUES LESS THAN (1991), PARTITION p1 VALUES LESS THAN (1996), PARTITION p2 VALUES LESS THAN (2001), PARTITION p3 VALUES LESS THAN MAXVALUE);
> show create table tp14;
+-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                              |
+-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| tp14  | CREATE TABLE `tp14` (
`id` INT NOT NULL,
`fname` VARCHAR(30) DEFAULT NULL,
`lname` VARCHAR(30) DEFAULT NULL,
`hired` DATE NOT NULL,
`separated` DATE NOT NULL,
`job_code` INT DEFAULT NULL,
`store_id` INT DEFAULT NULL
) |
+-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.01 sec)

> CREATE TABLE tp15 (a INT NOT NULL, b INT NOT NULL) PARTITION BY RANGE COLUMNS(a,b) PARTITIONS 4 (PARTITION p0 VALUES LESS THAN (10,5), PARTITION p1 VALUES LESS THAN (20,10), PARTITION p2 VALUES LESS THAN (50,20), PARTITION p3 VALUES LESS THAN (65,30));
> show create table tp15;
+-------+------------------------------------------------------------+
| Table | Create Table                                               |
+-------+------------------------------------------------------------+
| tp15  | CREATE TABLE `tp15` (
`a` INT NOT NULL,
`b` INT NOT NULL
) |
+-------+------------------------------------------------------------+
1 row in set (0.00 sec)

> CREATE TABLE tp16 (id   INT PRIMARY KEY, name VARCHAR(35), age INT unsigned) PARTITION BY LIST (id) (PARTITION r0 VALUES IN (1, 5, 9, 13, 17, 21), PARTITION r1 VALUES IN (2, 6, 10, 14, 18, 22), PARTITION r2 VALUES IN (3, 7, 11, 15, 19, 23), PARTITION r3 VALUES IN (4, 8, 12, 16, 20, 24));
show create table tp16;
> show create table tp16;
+-------+-------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                        |
+-------+-------------------------------------------------------------------------------------------------------------------------------------+
| tp16  | CREATE TABLE `tp16` (
`id` INT DEFAULT NULL,
`name` VARCHAR(35) DEFAULT NULL,
`age` INT UNSIGNED DEFAULT NULL,
PRIMARY KEY (`id`)
) |
+-------+-------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

> CREATE TABLE tp17 (id   INT, name VARCHAR(35), age INT unsigned) PARTITION BY LIST (id) (PARTITION r0 VALUES IN (1, 5, 9, 13, 17, 21), PARTITION r1 VALUES IN (2, 6, 10, 14, 18, 22), PARTITION r2 VALUES IN (3, 7, 11, 15, 19, 23), PARTITION r3 VALUES IN (4, 8, 12, 16, 20, 24));
> show create table tp17;
+-------+-----------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                    |
+-------+-----------------------------------------------------------------------------------------------------------------+
| tp17  | CREATE TABLE `tp17` (
`id` INT DEFAULT NULL,
`name` VARCHAR(35) DEFAULT NULL,
`age` INT UNSIGNED DEFAULT NULL
) |
+-------+-----------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

> CREATE TABLE tp18 (a INT NULL,b INT NULL) PARTITION BY LIST COLUMNS(a,b) (PARTITION p0 VALUES IN( (0,0), (NULL,NULL) ), PARTITION p1 VALUES IN( (0,1), (0,2), (0,3), (1,1), (1,2) ), PARTITION p2 VALUES IN( (1,0), (2,0), (2,1), (3,0), (3,1) ), PARTITION p3 VALUES IN( (1,3), (2,2), (2,3), (3,2), (3,3) ));
> show create table tp18;
+-------+--------------------------------------------------------------------+
| Table | Create Table                                                       |
+-------+--------------------------------------------------------------------+
| tp18  | CREATE TABLE `tp18` (
`a` INT DEFAULT NULL,
`b` INT DEFAULT NULL
) |
+-------+--------------------------------------------------------------------+
1 row in set (0.00 sec)
```

- Example 5

```sql
> drop table if exists t1;
> create table t1(a bigint primary key auto_increment,
    b varchar(10));
> insert into t1(b) values ('bbb');
> insert into t1 values (3, 'ccc');
> insert into t1(b) values ('bbb1111');
> select * from t1 order by a;
+------+---------+
| a    | b       |
+------+---------+
|    1 | bbb     |
|    3 | ccc     |
|    4 | bbb1111 |
+------+---------+
3 rows in set (0.01 sec)

> insert into t1 values (2, 'aaaa1111');
> select * from t1 order by a;
+------+----------+
| a    | b        |
+------+----------+
|    1 | bbb      |
|    2 | aaaa1111 |
|    3 | ccc      |
|    4 | bbb1111  |
+------+----------+
4 rows in set (0.00 sec)

> insert into t1(b) values ('aaaa1111');
> select * from t1 order by a;
+------+----------+
| a    | b        |
+------+----------+
|    1 | bbb      |
|    2 | aaaa1111 |
|    3 | ccc      |
|    4 | bbb1111  |
|    5 | aaaa1111 |
+------+----------+
5 rows in set (0.01 sec)

> insert into t1 values (100, 'xxxx');
> insert into t1(b) values ('xxxx');
> select * from t1 order by a;
+------+----------+
| a    | b        |
+------+----------+
|    1 | bbb      |
|    2 | aaaa1111 |
|    3 | ccc      |
|    4 | bbb1111  |
|    5 | aaaa1111 |
|  100 | xxxx     |
|  101 | xxxx     |
+------+----------+
7 rows in set (0.00 sec)
```
