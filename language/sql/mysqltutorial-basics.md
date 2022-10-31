Source: https://www.mysqltutorial.org/mysql-basics/

## Intro
- MySQL is a popular RDBMS with client-server model. Communicate through SQL. Alternative: PostgreSQL, Microsoft SQL Server
- SQL can instruct to query, manipulate data, data identity (schema/type), data access control
- **execution order:**
	- from -> where -> group by -> having -> select -> distinct -> order by (can use alias from select)

## MISC
- `FIELD(str, str1, str2...)` function: acceps a list of arguments, and returns the position of str in (str1, str2, ...). I.e. useful to map each column values to a number, and use that as the ORDER BY value
- `LIMIT x OFFSET y` for paginating
- `SHOW PROCESSLIST` to find out info/state of each session (ie. if their in the middle of pending query, sleep, etc.)


## Basics clauses
- WHERE 
  - combine expressions using AND, OR, NOT
  - `where col1 BETWEEN 1 and 5` - the BETWEEN operator returns true if col value in range
  - `where col1 LIKE '%son'` - LIKE operator returns true if match pattern. `%` wildcard matches string >=0 characters. `_` wildcard matches 1 char.
  - `where col1 IN (val1, val2, val3..)` - the IN operator returns true if value in list
  - `where col IS NULL` - if value is null
  - `where col1 = 'sales'`. Other comparison operator: (`<>` or `!=`), (`<` for date/time/numeric), (`<=`)
- DISTINCT
  - `SELECT DISTINCT (cols..)` determine the uniqueness of the result rows
- ORDER BY
  - SELECT returns rows in unspecified order. Use ORDER BY for deterministic result.
- INSERT
	- `INSERT INTO <table>(cols... ) VALUES (row1Vals...), (row2Vals...)`
	- `INSERT INTO SELECT ()`
		- can insert many rows without limit.
		- useful to copy / summarize (aggregate) data from other tables
	- `INSERT INTO ... ON DUPLICATE KEYS UPDATE <col1=val1>, <col2=val2>,..`
		- the assignments will trigger as a fallback when it tries to insert and a duplicate error is thrown. It updates the current existing record, instead of creating new one. (like try insert else update)
	- `INSERT IGNORE INTO ...`
		- ignores the row when error occurs, and proceed with inserting the other rows.
		- might also truncate values if too long for length check
- UPDATE
	- `UPDATE <table> SET <col1> = <val1>, <col2> = <val2>, .... WHERE ...`
		- if where is omitted, it'll update all rows in table (!)
		- similar to insert, there is `UPDATE IGNORE ... ` modifier
	- `UPDATE <table1> <table2> SET ....`
		- under the hood joins the 2 tables on the id col
		- or more generally `UPDATE <table1> <table2> [LEFT | INNER] JOIN <table1> ON <table1.col> = <table2.col> SET ....`
- DELETE
	- `DELETE FROM <table> WHERE ...`
		- if WHERE is omitted, it deletes all rows from data
	- use `TRUNCATE <table>` for better performance
		- doesn't return rows deleted
	- if foreign key constrained present, use `ON DELETE CASCADE` to auto delete the referenced rows in the child tables
	- `DELETE <table1> <table2> FROM <table1> JOIN <table2> ON ... WHERE...`
		- delete data from multiple tables
		- if DELETE <table1> FROM, it only deletes from table 1
- REPLACE
	- `REPLACE INTO <table>(cols... ) VALUES (row1Vals...), (row2Vals...)` replaces entire row if not exist (set non-specified col to null)
		- like insert, but if there's duplicate error it replaces.
	- `REPLACE INTO <table> SET ...`
		- like update without where clause, and uses default value if null
- PREPARE statement
	- in the past, you send sql string to server, and server needs to parse it. Gets expensive if you send the same query multiple times. So idea is send templated query once, execute many times with different values, and deallocate the prepared statement when done
	- `PREPARE <statement name> FROM <ie. select query string where col = ?>`
	- then you can repeatedly do:
		- SET @x = 'value'
		- EXECUTE <statement name> USING @x
- CREATE db
	- `CREATE DATABASE [IF NOT EXISTS] <db name> [CHARACTER SET charset_name] [COLLATE <collation_name>]`
- DROP db
	- delete db and all its tables
- MySQL 5.7 introduced 'generated column'. 
	- `CREATE TABLE <name> (... <column name> <data type> GENERATED ALWAYS AS <i.e. concat first and last names>... )`
	- GENERATED ALWAYS AS <expression> - defines a column with generated values. Computed on the fly when querying the table (virtual column)

## Aliases
- column alias - can be used in (ORDER BY, HAVING, GROUP BY). Not in WHERE clause
- table alias can be used anywhere since FROM executes first

## Logical operators
- values: 0, 1, NULL
- AND precedes OR (use paranthesis)
- `IN` (or `NOT IN`) used in WHERE, UPDATE, DELETE, subquery
- `val BETWEEN low AND high` is 1 when high >= val >= low
  - for dates, you need to cast it. `BETWEEN CAST('2003-01-01' AS DATE) AND ...`
  - CAST -> convert string literal to date
  - `NOT BETWEEN`
- `<val> LIKE <pattern> ` (or `NOT LIKE`)

## JOINS
- Inner Join:
  - compare each row from table A and table B (O(n^2)). If pair match join predicate,  it creates row in the result set.
  - `INNER JOIN .. USING (col)` is equivalent to `INNER JOIN ... ON a.col = b.col`
- Left join:
  - if join predicate don't match, it creates row for table A's fields and NULL for B's fields. It'll always select row from left table, whether or not there's matching row in right table.
  - to find row in A that's not in B, you can do `A LEFT JOIN B USING(col) WHERE B.id IS NULL`
  - ie. customers and their orders (whether or not there's any)
  - ie. employee and their (>=0) customer, and their (>=0 payments)
- right join
- cross join
  - don't have join condition
  - the complete cross product set of A's rows and B's rows
  - cross join + where == inner join
  - i.e. you can left join A with a cross join of B*C, to get the combination that leads to A=null
- self join (join with same table + alias)
  - used to query heirarchial data / compare rows within the same table
  - ie. employee has reports_to column to reflect org chart. When you self-join, table A can refer to 'managers' and table B can refer to 'direct report'

## Grouping data
- GROUP BY
  - aggregate/reduce each group of rows into 1 row.. Group by an expression or col values. Once grouped, you can use select with aggregation like sum, avg, min, max, count...
	- you could use expression to group. Ie. `SELECT YEAR(dateCol), <aggregate>.. GROUP BY YEAR(dateCol)`
	- without aggregate functions, they behave like distinct
- Having
  - to filter groups returned by group-by (without group-by it behaves like where)
	- ie. `GROUP BY year HAVING year > 2003`
	- note: although 'having' is before 'select', it can use the aggregated alias from select
	- conditions on aggregated value, instead of row values like where. (ie. x this month, year..)
- ROLLUP
  - generate subtotal and total of the group by
	- grouping set is the set of columns that you group-by. Sometimes you want to show aggregation of the aggregation, which you could do with Union (but inefficient).
	- rollup lets you do aggregation of groups (basically running the query on every subset of the grouping set - including null set)
	- GROUPING(col) - is a helper to identify if its a rollup row

## Subqueries
- Subquery
	- inner-query can be used anywhere an expression is used.
	- subquery in WHERE. If subquery return 1 val, you can use <, =, >... If subquery return list of values, you could use IN and NOT IN.
		- ie. `WHERE code in (select all code from another table)`
		- ie. `WHERE amount = (select MAX(amount) form payment)` to return one row result. Or `WHERE amount > (select avg amount)`
	- subquery in FROM. Outer-query can access all columns from inner-query. Inner-query is saved in temporary table (aka derived/materialized table)
	- correlated subquery
		- subqueries that run per row. If innerquery references the outerquery's column
		- ie. `WHERE buyPrice > (select avg buyPrice where/for outerTable.productline)`
		- ie. `SELECT customer... WHERE EXISTS(subquery that uses the current row customer)` to filter customer. Exists == true when subquery returns anything (the filter). Might be more efficient than join to do these n+1 queries
- Derived table is a virtual table (used in SELECT .. FROM <subtable><alias>)
	- Simpler than temp table since it's doesn't create the table
	- Unlike stand-alone sub-query (that can run independently with outer table), a derived table requires an alias, so you can ref it later
	- ie. `Select <subquery that filters out column> inner join <table>`: might be more efficient than inner join the whole tables.
- subquery on EXISTS vs IN (used as condition to select/update/delete)
	- rule-of-thumb: Use EXISTS when subquery is expensive. Use IN when subquery is simple.
	- `WHERE EXISTS(<correlated subquery to filter>)` stop scanning subquery when result is found. Whereas `WHERE col IN <subquery>` query only once but scans the whole table so more expensive if large.
- EXPLAIN gets the query cost and path diagram to analyze performance
  
## CTE
- Common Table Expression (CTE)
	- like Derived table, it only lasts on query runtime and not stored as obj.
	- Unlike Derived table, CTE can be recursive or referenced multiple times in the query. Better readability & performance
	- `WITH <cte_name> AS ( <query> ), <cte_name2> AS (..)...`. You can use the cte like a table after its defined.
- Recursive CTE (the beast) to traverse heirarchial data
	- `WITH RECURSIVE <cte_name> AS ( select 1 UNION ALL select n+1 from <cte_name> where <termination condition n<3>)`
	- you have a base condition UNION ALL the recursive set which must have termination and call the recursive cte with whatever that it's currently returning (n+1).
  
## Set operator
- Union
  - combine select statements set
	- condition: #, type, and order of col of select statement should be the same
	- `UNION DISTINCT` by default. Clears any duplicate of 2 sets
	- `UNION ALL` don't remove duplicate
	- uses the name of the first select, and you can add 'order by' at the end of all unions.
- Minus
  - set minus: in mysql you can use left join and where not null

## Transaction
- to make sure db never contains partial operation (all or none)
- `start transaction`, `commit`, `rollback`. `set autocommit = on/off` (on default)
- Table locking:
  - client session can reserve locks
  - read lock: any number can acquire the lock and read at the same time, but none can write until all the read lock is released
  - write lock: those who require it can read/write, others can't
	- `LOCK TABLES <tbl_name> [READ | WRITE]` to acquire lock
	- `UNLOCK TABLES;` to release
	- when read lock is held: others can read without a read lock, but none can write until its released. The lock holder will error if it tries to write. The non-lock holders will have their operation in 'waiting' state (and execute when released)
	- when write lock is held: only holder can r/w. Others cant r & w (they'll be put in waiting state)
  
## Data types
- identified by kind of values, space it takes up (fix/variable length), can it be indexed, how the values are compared
- numeric data types
  - tinyint, smallint, mediumint, int, bigint, decimal (fix point), float (single-precision floating point), double(double-precision floating point), bit
  - BIT(n)
    - insert string literal in the form of `0b'<binary string>'` or `b<binary string>`.
    - querying it will return integer, so if you want it to display the binary string you need `bin(<col name>)`.
    - use lpad(bin(<col name>)) to left pad the string till length n
  - Decimal(<precision/s.d.(total # digits)>, <# digits after decimal point>)
    - alias of DEC, FIXED, NUMERIC. Can have unsigned
    - use: for monetary values
    - whole number: DECIMAL(P) or DECIMAL(P,0)
    - 4 bytes for every 9 digits
- boolean
  - no built-in boolean. To represent, it uses smallest int type under the hood outside the col definition (TINYINT(1))
  - to test a boolean value in where clause, use `IS FALSE`/`IS NOT TRUE`, `IS TRUE`
- string (plain text, binary data)
  - can be searched with LIKE/regex/full-text search #toread
  - CHAR (fix length char/non-binary string), VARCHAR (variable length char string), BINARY (fix length binary string), VARBINARY, TINYBLOB (tiny blob/binary-large-object), BLOB, MEDIUMBLOB, LONGBLOB, TINYTEXT (tiny non-binary string), TEXT (small non-binary string), MEDIUMTEXT, LONGTEXT, ENUM, SET
  - char(n: 0-255)
    - if fix size, you get better performance than varchar
    - Pads with spaces until length n. On query (and even when using LENGTH() function), it removes the trailing spaces. If you insert trailing spaces manually, it also removes them.
    - comparing ignores the trailing spaces. But using LIKE consider the trailing spaces.
  - enum
    - compact. Stores numeric indexes to represent string. Mysql maps them under the scenes - users don't need to know number stored (they can either insert/query with numeric/string value)
    - e.g. `<col_name> ENUM('val1', 'val2', ...)`
      - define the order to match how you want it sorted
  - Text
    - 1byte to 4 GB of text (ie. article body, product description)
    - not stored in memory, but the disk. It'll be slower than querying varchar/char
    - variant: TinyText (255 bytes/char max) (ie. summary)
    - variant: Text (up to 64 KB) (ie. article)
    - variant: MediumText (16MB) (ie. text of book, white papers..)
    - variant: LongText (4GB)
  - Varchar
    - 1-2 byte length prefix (overhead) + actual data
    - size limited by the max row size (size of all column in the record)
- date/time
  - DATE, TIME, DATETIME, TIMESTAMP, YEAR
  - DATETIME: 'YYYY-MM-DD HH:MM:SS'
    - 5 bytes storage
  - TIMESTAMP is a bit smaller, requires 4 bytes (but is designed to range from 1970-2038). Timestamp is stored in UTC, while datetime stores itself as it is without timezone.
    - with timestamp, querying in 2 different timezones will give you different results (not the utc value)
    - with datetime, you need to handle timezone in app code (which might be better...)
  - `NOW()` generates datetime value in the timezone set up for the session (you can change by `SET time_zone = '+03:00`)
:logbook:
          CLOCK: [2022-10-15 Sat 18:13:35]
:END:
  - `DATE(NOW())` generates date portion of datetime (yyyy-mm-dd)
    - useful for querying datetime with just date portion (`SELECT ... WHERE DATE(created_at) = '2015-11-05'`)
  - `TIME(NOW())` generates time portion (hh:mm:ss)
  - hour(), minute(), second, day, week, month, quarter, year
  - `DATE_FORMAT(NOW(), '%W %M...')` to format -> 'Thursday November'
  - `DATE_ADD(NOW(), INTERVAL 1 MINUTE)` (or DATE_SUB) to add interval to datetime value
  - `DATEDIFF(NOW(), NOW())` returns # days between 2 datetime
- spatial data
  - GEOMETRY (spatial val of any type), POINT (x-y coordinate), LINESTRING (curve with many point values), POLYGON, GEOMETRYCOLLECTION (group of geometry), MULTISTRING (group of linestring), MULTIPOINT (collection of point), MULTIPOLYGON
- JSON data
  - since v5.7.8. Autovalidate
  - allow to query values within json by key/index
  - cannot have default value, nor indexed. (you can have an index on 'generated column' extracted from the JSON col)
  - to query json columns, use 'column-path' operator `->`, or inline path operator `- > >` similar but values won't have quotation surrounding it
  - `SELECT <json_col>->'$.name'`

## Tables
- ALTER TABLE
	- add column (`ALTER TABLE <name> ADD <col def>`),
	- alter column (`ALTER TABLE <name> MODIFY <col def>`),
	- rename column (`ALTER TABLE <name> CHANGE COLUMN <oldname> <newname> <column def>`), (similar to `RENAME TABLE` command)
	- drop column (`ALTER TABLE <name> DROP COLUMN <name>`),
	- rename table (`ALTER TABLE <name> RENAME TO <new_name>`)
	- careful dropping/renaming tables as you'll also need to modify all the references in views, stored procedure, foreign keys, triggers, app code..
- DROP TABLE
	- security risk: if you drop a table it doesn't remove the user privileges. So recreating the table after would apply existing privileges.
- Temporary tables
	- store result in a table you can reuse many times in the session (when query is too expensive to store in one select)
	- `CREATE TEMPORARY TABLE` (syntax is like `create table`)
	- either create table and insert into, OR,
	-
	  ``` SQL
	  	  CREATE TEMPORARY TABLE <table_name>
	  	  	SELECT ... FROM ...;
	  ```
- CREATE TABLE
	- ` CREATE TABLE <table_name> (column definitions, ..., table constraints, ...) ENGINE=<innodb by default>`
	- column definition: `<col name> <data_type(len)> [not null] [default val] [auto_increment] <column_constraint>`
	- table constraints: (e.g. unique, check, primary key, foreign key)
		- i.e. `PRIMARY KEY (c1, c2)` to constraint a set of columns
		- i.e. `FOREIGN KEY(c1) REFERENCES <table>(col) ON UPDATE RESTRICT ON DELETE CASCADE`
	- `DESCRIBE <table_name>` to see table structure
- SEQUENCE
	- generate random unique ID (set column as AUTO_INCREMENT)
	- +1 if you insert a NULL. You can write value to it too (it'll error if not unique) - which sets it as new starting sequence for next generation, and make gaps in the sequence. BUT if you 'update', it won't set that as a starting sequence (updating 3 to 10 will still generate 4 for next insert)
	- `LAST_INSERT_ID()` to get last insert id
- column constraints
	- NOT NULL
		- good practice to NOT NULL by default unless there's a reason. It makes query more complicated to handle the null cases.
	- Primary key(s)
		- must be integers (INT/BIGINT). Sufficient to store all rows the table has.
	- Unique key
		- like primary, but allows null
		- `ALTER TABLE <tablename> ADD UNIQUE INDEX <index name> (<col name> ASC)`
	- Foreign key
		- e.g. customers & orders is 1:many. The orders have a foreign key customerId. customer is the parent/referenced table, while order is child/referencing table.
		- can have >1 foreign key, typically each referencing to a PK
		- self-referencing FK (ie. reporting heirarchy where reportsTo reference pk)
		- in create/alter table, `FOREIGN KEY <fk name> (<col_name_list>) REFERENCES <parent_table>(<col_name_list>) [ON DELETE <reference_option>] [ON UDPATE reference_option]`
		- fk_name is optional. Will be autogenerated if omitted
		- <reference_option> is actions mysql will take when referred parent key col is deleted/updated. Options are CASCADE (child is delete/update with its parent), SET NULL (child set to null on parent delete/update), RESTRICT (don't allow parent to delete/update if there's a matching row) (default)
	- `SHOW CREATE TABLES <table_name>` or `SHOW INDEX FROM <table_name>`
		- show definition of a table (useful to find the index name)
	- CHECK constraint
		- ensure col value satisfy a bool expression
		- `CHECK(<expression>)` (e.g. cost >= 0, or a table constraint like price >= cost)
	- `DEFAULT <literal constant>` constraint
- `LOAD DATA INFILE`: to import csv to table
	- `LOAD DATA [LOCAL] INFILE <path> INTO TABLE <tbl_name> FIELDS TERMINATED BY ',' ENCLOSED BY '"' LINES TERMINATED BY '\n' IGNORE 1 ROWS`
	- you can add lines to transform data before they're inserted
	- use fields enclosed by, so ',' inside "" isn't recognized as field separators
	- adding local will send the file from local to server (might take some time. might have some security issues)
- export table to csv
	- `<select query> INTO OUTFILE <path> FIELDS ENCLOSED BY...`
	- use union to prepend column headings
	- will set 'N' for null values. Use IFNULL(<col>, <default>) to customize it
	- idea: backup?
  
## Globalization
- character set:
	- all letters & their encodings
	- default is `latin1`. For multi-language column use unicode (ie. utf8)
- collation:
	- rules to compare characters in character set (analogy: like equivalent check). A char set have >=1 collation, but 2 char set can't have same collation.
	- e.g. `latin1_sweidsh_ci`. ci for case insensitive
	- can be defined in the server/table/column level
  
## mysql client
- `mysql -u root -p` login with root user account
- `SHOW DATABASES;` list available database you can run the `USE <db name>` command
- `SELECT database()` to get current db
