# SQL - SELECT

Orient supports the SQL language to execute queries against the database engine. Take a look at the [operators](SQL-Where.md#operators) and [Functions](SQL-Where.md#functions). To learn the main differences in comparison to the SQL-92 standard, take a look at: [OrientDB SQL](SQL.md).

# Syntax
```sql
SELECT [<Projections>] [FROM <Target> [LET <Assignment>*]]
    [WHERE <Condition>*]
    [GROUP BY <Field>*]
    [ORDER BY <Fields>* [ASC|DESC] *]
    [UNWIND <Field>*]
    [SKIP <SkipRecords>]
    [LIMIT <MaxRecords>]
    [FETCHPLAN <FetchPlan>]
    [TIMEOUT <Timeout> [<STRATEGY>]
    [LOCK default|record]
    [PARALLEL]
    [NOCACHE]
```

- **[Projections](SQL-Query.md#projections)**, optionally, is the data you want to extract from the query as the result set. Look at [Projections](SQL-Query.md#projections).
- **Target** can be a class, cluster, single [RID](Concepts.md#recordid), set of [RID](Concepts.md#recordid)s or index values sorted by ascending or descending key order (index values were added in 1.7.7). **Class** is the class name on which to execute the query. Similarly, specifying **cluster** with the `cluster:` prefix executes the query within that cluster only. You can fetch records not from a cluster but instead from an index using the following prefixes: `indexvalues:`, `indexvaluesasc:` or `indexvaluesdesc:`. If you are using `indexvalues:` or `indexvaluesasc:` prefix records will be sorted in ascending order of index keys. If you are using  `indexvaluesdesc:` prefix records will be sorted in descending order of index keys. Use one or more [RID](Concepts.md#recordid)s to specify one or a small set of records. This is useful in order to specify a starting point when navigating graphs.
- **[WHERE](SQL-Where.md)** condition is common to the other SQL commands and is described in a dedicated section of the documentation.
- **[LET](SQL-Query.md#let-block)** is the part that binds context variables to be used in projections, conditions or sub-queries
- **GROUP BY** is in accordance to the standard SQL syntax specifying the field to perform the grouping. The current release supports only 1 field.
- **ORDER BY** is in accordance to the standard SQL syntax specifying fields with an optional ASC or DESC (default is ASCending). If you are using a projection in your query, ensure the ORDER BY field is included in this projection.
- **[UNWIND](SQL-Query.md#unwind)** (since 2.1) allows to unwind collection fields
- **SKIP** skips `<SkipRecords>` the specified number of records starting at the beginning of the result set. This is useful for [Pagination](Pagination.md) when used in conjunction with `LIMIT`.
- **LIMIT** sets the maximum number of records returned by the query to `<MaxRecords>`. This is useful for [Pagination](Pagination.md) when used in conjunction with SKIP.
- **FETCHPLAN** sets the [fetchplan](Fetching-Strategies.md). Example: `FETCHPLAN out:3` to pre-fetch up to 3rd level under `out` field.
- **TIMEOUT** sets the maximum timeout in milliseconds for the query. By default the query has no timeout. If you don't specify the strategy, the default strategy `EXCEPTION` is used. Strategies are:
 - `RETURN`, truncate the result set returning the data collected up until the timeout
 - `EXCEPTION`, default one, throws an exception if the timeout has been reached
- **LOCK** manage the locking strategy. By default is "default", that means release the lock once the record is read, while "record" means to keep the record locked in exclusive mode in current transaction till the transaction has been finished by a commit or rollback operation.
- **PARALLEL** execute the query against X concurrent threads, where X is the number of processors/cores found on the host OS of the query. PARALLEL execution is useful on long running queries or queries that involve multiple clusters. On simple queries using PARALLEL could cause a slow down due to the overhead inherent with using multiple threads
- **NOCACHE** avoids using the cache

*NOTE: Starting from 1.0rc7 the `RANGE` keyword has been removed. To execute range queries use the `BETWEEN` operator against `@rid` as explained  in [Pagination](Pagination.md).*

## Projections

In standard SQL, projections are mandatory. In OrientDB, if it's omitted, the entire record set is returned. It is the equivalent of the `*` keyword. Example:
```sql
SELECT FROM Account
```

With all projections except the wildcard "*", a new temporary document is created and the `@rid` and `@version` of the original record will not be included.

```sql
SELECT name, age FROM Account
```

The conventional naming for the returned document's fields are:
- the field name for plain fields `invoice -> invoice`
- the first field name for chained fields, like `invoice.customer.name -> invoice`
- the name of the function for functions, like `max(salary) -> max`

If the target field already exists, a progressive number is used as a prefix. Example:
```sql
SELECT max(incoming), max(cost) FROM Balance
```

Will return a document with the field `max` and `max2`.

To override the field name, use `AS`. Example:
```sql
SELECT max(incoming) AS max_incoming, max(cost) AS max_cost FROM Balance
```

By using the dollar (`$`) as a prefix, you can access context variables. Each time you run a command, OrientDB accesses the context to read and write variables. Here's an example to display the path and depth level of the [traversal ](SQL-Traverse.md) on all the movies, up to the 5th level of depth:
```sql
SELECT $path, $depth FROM ( TRAVERSE * FROM Movie WHERE $depth <= 5 )
```
# Examples

Get all the records of type `Person` where the name starts with `Luk`:
```sql
SELECT * FROM Person WHERE name LIKE 'Luk%'
```
or
```sql
SELECT * FROM Person WHERE name.left(3) = 'Luk'
```
or
```sql
SELECT * FROM Person WHERE name.substring(0,3) = 'Luk'
```

Get all the records of type `!AnimalType` where the collection `races` contains at least one entry where the first character of the name, ignoring the case, is equal to `e`:
```sql
SELECT * FROM animaltype WHERE races CONTAINS(name.toLowerCase().subString(0,1) = 'e')
```

Get all the records of type `!AnimalType` where the collection `races` contains at least one entry with name `European` or `Asiatic`:
```sql
SELECT * FROM animaltype WHERE races CONTAININS(name in ['European','Asiatic'])
```

Get all the records of type `Profile` where any field contains the word `danger`:

```sql
SELECT FROM profile WHERE any() LIKE '%danger%'
```

Get any record at any level that has the word `danger`:
```sql
SELECT FROM profile WHERE any() TRAVERSE( any() LIKE '%danger%' )
```

Get all the records where up to the 3rd level of connections has some field that contains the word `danger` ignoring the case:
```sql
SELECT FROM Profile WHERE any() TRAVERSE( 0,3 ) ( any().toUpperCase().indexOf( 'danger' ) > -1 )
```

Order the result set by `name` in descending order:
```sql
SELECT FROM Profile ORDER BY name DESC
```

Return the total of records per city:
```sql
SELECT SUM(*) FROM Account GROUP BY city
```

Traverse records starting from a root node:

```sql
SELECT FROM 11:4 WHERE any() TRAVERSE(0,10) (address.city = 'Rome')
```

Query only a set of records:
```sql
SELECT FROM [#10:3, #10:4, #10:5]
```

Select only three fields from Profile:
```sql
SELECT nick, followings, followers FROM Profile
```

Select the `name` field in upper-case and the `country name` of the linked city of the address:
```sql
SELECT name.toUppercase(), address.city.country.name FROM Profile
```

Order by record creation. Starting from 1.7.7, using the expression "order by @rid desc", allows OrientDB to open an Inverse cursor against clusters. This is extremely fast and doesn't require classic ordering resources (RAM and CPU):
```sql
SELECT FROM Profile ORDER BY @rid DESC
```

## LET block

The `LET` block contains the list of context variables to assign each time a record is evaluated. These values are destroyed once the query execution ends. Context variables can be used in projections, conditions, and sub-queries.

## Assign fields to reuse multiple times

OrientDB allows crossing relationships, but if in a single query you need to evaluate the same branch of the nested relationship, it's definitely better using a context variable that refers to the full relationship.

Example:
```sql
SELECT FROM Profile
WHERE address.city.name like '%Saint%"' and
    ( address.city.country.name = 'Italy' or address.city.country.name = 'France' )
```

Using LET becomes shorter and faster, because the relationships are traversed only once:
```sql
SELECT FROM Profile
LET $city = address.city
WHERE $city.name like '%Saint%"' AND
    ( $city.country.name = 'Italy' OR $city.country.name = 'France' )
```

In this case the path till `address.city` is traversed only once.

## Sub-query

LET block allows you to assign a context variable to the result of a sub-query. Example:
```sql
SELECT FROM Document
LET $temp = (
    SELECT @rid, $depth FROM (
        TRAVERSE V.out, E.in FROM $parent.current
    )
    WHERE @class = 'Concept' AND (id = 'first concept' OR id = 'second concept' )
)
WHERE $temp.size() > 0
```

## Usage in projection

Context variables can be part of result set used in [Projections](#Projections). The example below displays the city name of the previous example:
```sql
SELECT $temp.name FROM Profile
LET $temp = address.city
WHERE $city.name like '%Saint%"' AND
    ( $city.country.name = 'Italy' OR $city.country.name = 'France' )
```

## Unwind

(since 2.1)

Allows unwinding of collection fields and obtaining multiple records as a result, one for each element in the collection:

```sql
SELECT name, out("Friend").name as friendName from Person
```
The result will be something like
```
| name   | friendName         |
| 'John' | [ 'Mark', 'Steve'] |
```

If you want one record for each element in friendName, you can rewrite the query using UNWIND:

```sql
SELECT name, out("Friend").name as friendName from Person UNWIND friendName
```

and the result will be
```
| name   | friendName         |
| 'John' | 'Mark'             |
| 'John' | 'Steve'            |
```

# Conclusion

To know more about other SQL commands, take a look at [SQL commands](SQL.md).

## History
### 1.7.7
New targets `indexvalues:`, `indexvaluesasc:`, `indexvaluesdesc:` are added.
### 1.7
- Added **PARALLEL** keyword to execute the query against X concurrent threads, where X is the number of processors/core found on the os where the query is running (since 1.7). PARALLEL execution is useful on long running query or query that involve multiple clusters. On simple queries, using PARALLEL could cause a slow down because of the overhead on using multiple threads
