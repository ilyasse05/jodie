# jodie

This library provides helpful Delta Lake and filesystem utility functions.

![jodie](images/jodie.jpeg)

## Accessing the library

Fetch the JAR file from Maven.

```scala
libraryDependencies += "com.github.mrpowers" %% "jodie" % "0.0.3"
```

You can find the spark-daria releases for different Scala versions:

* [Scala 2.12 versions here](https://repo1.maven.org/maven2/com/github/mrpowers/jodie_2.12/)
* [Scala 2.13 versions here](https://repo1.maven.org/maven2/com/github/mrpowers/jodie_2.13/)

## Delta

### Type 2 SCDs

This library provides an opinionated, conventions over configuration, approach to Type 2 SCD management.  Let's look at an example before covering the conventions required to take advantage of the functionality.

Suppose you have the following SCD table with the `pkey` primary key:

```
+----+-----+-----+----------+-------------------+--------+
|pkey|attr1|attr2|is_current|     effective_time|end_time|
+----+-----+-----+----------+-------------------+--------+
|   1|    A|    A|      true|2019-01-01 00:00:00|    null|
|   2|    B|    B|      true|2019-01-01 00:00:00|    null|
|   4|    D|    D|      true|2019-01-01 00:00:00|    null|
+----+-----+-----+----------+-------------------+--------+
```

You'd like to perform an upsert with this data:

```
+----+-----+-----+-------------------+
|pkey|attr1|attr2|     effective_time|
+----+-----+-----+-------------------+
|   2|    Z| null|2020-01-01 00:00:00| // upsert data
|   3|    C|    C|2020-09-15 00:00:00| // new pkey
+----+-----+-----+-------------------+
```

Here's how to perform the upsert:

```scala
Type2Scd.upsert(deltaTable, updatesDF, "pkey", Seq("attr1", "attr2"))
```

Here's the table after the upsert:

```
+----+-----+-----+----------+-------------------+-------------------+
|pkey|attr1|attr2|is_current|     effective_time|           end_time|
+----+-----+-----+----------+-------------------+-------------------+
|   2|    B|    B|     false|2019-01-01 00:00:00|2020-01-01 00:00:00|
|   4|    D|    D|      true|2019-01-01 00:00:00|               null|
|   1|    A|    A|      true|2019-01-01 00:00:00|               null|
|   3|    C|    C|      true|2020-09-15 00:00:00|               null|
|   2|    Z| null|      true|2020-01-01 00:00:00|               null|
+----+-----+-----+----------+-------------------+-------------------+
```

You can leverage the upsert code if your SCD table meets these requirements:

* Contains a unique primary key column
* Any change in an attribute column triggers an upsert
* SCD logic is exposed via `effective_time`, `end_time` and `is_current` column

`merge` logic can get really messy, so it's easiest to follow these conventions.  See [this blog post](https://mungingdata.com/delta-lake/type-2-scd-upserts/) if you'd like to build a SCD with custom logic.

### Kill Duplicates

The function `killDuplicateRecords` deletes all the duplicated records from a table given a set of columns.

Suppose you have the following table:

```
+----+---------+---------+
|  id|firstname| lastname|
+----+---------+---------+
|   1|   Benito|  Jackson| # duplicate
|   2|    Maria|   Willis|
|   3|     Jose| Travolta| # duplicate
|   4|   Benito|  Jackson| # duplicate
|   5|     Jose| Travolta| # duplicate
|   6|    Maria|     Pitt|
|   9|   Benito|  Jackson| # duplicate
+----+---------+---------+
```

We can Run the following function to remove all duplicates:

```scala
DeltaHelpers.killDuplicateRecords(
  deltaTable = deltaTable, 
  duplicateColumns = Seq("firstname","lastname")
)
```

The result of running the previous function is the following table:

```
+----+---------+---------+
|  id|firstname| lastname|
+----+---------+---------+
|   2|    Maria|   Willis|
|   2|    Maria|     Pitt| 
+----+---------+---------+
```

### Remove Duplicates

The functions `removeDuplicateRecords` deletes duplicates but keeps one occurrence of each record that was duplicated.
There are two versions of that function, lets look an example of each,

#### Let’s see an example of how to use the first version:

Suppose you have the following table:

```
+----+---------+---------+
|  id|firstname| lastname|
+----+---------+---------+
|   2|    Maria|   Willis|
|   3|     Jose| Travolta|
|   4|   Benito|  Jackson|
|   1|   Benito|  Jackson| # duplicate
|   5|     Jose| Travolta| # duplicate
|   6|    Maria|   Willis| # duplicate
|   9|   Benito|  Jackson| # duplicate
+----+---------+---------+
```
We can Run the following function to remove all duplicates:

```scala
DeltaHelpers.removeDuplicateRecords(
  deltaTable = deltaTable,
  primaryKey = "id",
  duplicateColumns = Seq("firstname","lastname")
)
```

The result of running the previous function is the following table:

```
+----+---------+---------+
|  id|firstname| lastname|
+----+---------+---------+
|   2|    Maria|   Willis|
|   3|     Jose| Travolta|
|   4|   Benito|  Jackson| 
+----+---------+---------+
```

#### Now let’s see an example of how to use the second version:

Suppose you have a similar table:

```
+----+---------+---------+
|  id|firstname| lastname|
+----+---------+---------+
|   2|    Maria|   Willis|
|   3|     Jose| Travolta| # duplicate
|   4|   Benito|  Jackson| # duplicate
|   1|   Benito|  Jackson| # duplicate
|   5|     Jose| Travolta| # duplicate
|   6|    Maria|     Pitt|
|   9|   Benito|  Jackson| # duplicate
+----+---------+---------+
```

This time the function takes an additional input parameter, a primary key that will be used to sort 
the duplicated records in ascending order and remove them according to that order.

```scala
DeltaHelpers.removeDuplicateRecords(
  deltaTable = deltaTable,
  primaryKey = "id",
  duplicateColumns = Seq("firstname","lastname")
)
```

The result of running the previous function is the following:

```
+----+---------+---------+
|  id|firstname| lastname|
+----+---------+---------+
|   1|   Benito|  Jackson|
|   2|    Maria|   Willis|
|   3|     Jose| Travolta|
|   6|    Maria|     Pitt|
+----+---------+---------+
```

These functions come in handy when you are doing data cleansing.

### Copy Delta Table

This function takes an existing delta table and makes a copy of all its data, properties,
and partitions to a new delta table. The new table could be created based on a specified path or
just a given table name. 

Copying does not include the delta log, which means that you will not be able to restore the new table to an old version of the original table.

Here's how to perform the copy to a specific path:

```scala
DeltaHelpers.copyTable(deltaTable = deltaTable, targetPath = Some(targetPath))
```

Here's how to perform the copy using a table name:

```scala
DeltaHelpers.copyTable(deltaTable = deltaTable, targetTableName = Some(tableName))
```

Note the location where the table will be stored in this last function call 
will be based on the spark conf property `spark.sql.warehouse.dir`.

### Latest Version of Delta Table
The function `latestVersion` return the latest version number of a table given its storage path. 

Here's how to use the function:
```scala
DeltaHelpers.latestVersion(path = "file:/path/to/your/delta-lake/table")
```

### Insert Data Without Duplicates
The function `appendWithoutDuplicates` inserts data into an existing delta table and prevents data duplication in the process.
Let's see an example of how it works.

Suppose we have the following table:

```
+----+---------+---------+
|  id|firstname| lastname|
+----+---------+---------+
|   1|   Benito|  Jackson|
|   4|    Maria|     Pitt|
|   6|  Rosalia|     Pitt|
+----+---------+---------+
```
And we want to insert this new dataframe:

```
+----+---------+---------+
|  id|firstname| lastname|
+----+---------+---------+
|   6|  Rosalia|     Pitt| # duplicate
|   2|    Maria|   Willis|
|   3|     Jose| Travolta|
|   4|    Maria|     Pitt| # duplicate
+----+---------+---------+
```

We can use the following function to insert new data and avoid data duplication:
```scala
DeltaHelpers.appendWithoutDuplicates(
  deltaTable = deltaTable,
  appendData = newDataDF, 
  compositeKey = Seq("firstname","lastname")
)
```

The result table will be the following:

```
+----+---------+---------+
|  id|firstname| lastname|
+----+---------+---------+
|   1|   Benito|  Jackson|
|   4|    Maria|     Pitt|
|   6|  Rosalia|     Pitt|
|   2|    Maria|   Willis|
|   3|     Jose| Travolta| 
+----+---------+---------+
```
### Generate MD5 from columns

The function `withMD5Columns` appends a md5 hash of specified columns to the DataFrame. This can be used as a unique key 
if the selected columns form a composite key. Here is an example

Suppose we have the following table:

```
+----+---------+---------+
|  id|firstname| lastname|
+----+---------+---------+
|   1|   Benito|  Jackson|
|   4|    Maria|     Pitt|
|   6|  Rosalia|     Pitt|
+----+---------+---------+
```

We use the function in this way:
```scala
DeltaHelpers.withMD5Columns(
  dataFrame = inputDF,
  cols = List("firstname","lastname"),
  newColName = "unique_id")
)
```

The result table will be the following:
```
+----+---------+---------+----------------------------------+
|  id|firstname| lastname| unique_id                        |
+----+---------+---------+----------------------------------+
|   1|   Benito|  Jackson| 3456d6842080e8188b35f515254fece8 |
|   4|    Maria|     Pitt| 4fd906b56cc15ca517c554b215597ea1 |
|   6|  Rosalia|     Pitt| 3b3814001b13695931b6df8670172f91 |
+----+---------+---------+----------------------------------+
```

You can use this function with the columns identified in findCompositeKeyCandidate to append a unique key to the DataFrame.

### Find Composite Key
This function `findCompositeKeyCandidate` helps you find a composite key that uniquely identifies the rows your Delta table. 
It returns a list of columns that can be used as a composite key. i.e:

Suppose we have the following table:

```
+----+---------+---------+
|  id|firstname| lastname|
+----+---------+---------+
|   1|   Benito|  Jackson|
|   4|    Maria|     Pitt|
|   6|  Rosalia|     Pitt|
+----+---------+---------+
```

Now execute the function:
```scala
val result = DeltaHelpers.findCompositeKeyCandidate(
  deltaTable = deltaTable,
  excludeCols = Seq("id")
)
```

The result will be the following:

```scala
Seq("firstname","lastname")
```

### Delete rows from Deltatable where exists in an other Dataframe
The function `deleteFromAnotherDataframe` delete rows from existing delta table where rows exists in an another Dataframe.
Let's see an example of how it works.

Suppose we have the following delta table:

```
+----+---------+---------+----------+
|  id|firstname| lastname|      City|
+----+---------+---------+----------+
|   1|   Benito|  Jackson|     Paris| # this row will be deleted
|   2|    Maria|   Willis|    London|
|   3|     Jose| Travolta|    Mexico|
|   4|    Maria|     Pitt|    Madrid|
|   6|     Nora|   Fatehi| Marrakech|
+----+---------+---------+----------+
```
And we want to delete rows with (id and firstname) values equals to (id and firstname) in this dataframe:

```
+----+---------+---------+---------------------------------+
|  id|firstname| lastname| unique_id                       |
+----+---------+---------+---------------------------------+
|   1|   Benito|  Jackson| cad17f15341ed95539e098444a4c8050|
|   4|    Brad |   Willis| 3e1e9709234c6250c74241d5886d5073|
|   5|   George| Travolta| 1f1ac7f74f43eff911a92f7e28069271|
+----+---------+---------+---------------------------------+
```

We can use the following function to delete rows from delta Table where column id of deltaTable is equals to id of Dataframe and column firstname of deltTable is equals to firstname of Dataframe.
the attrColNameOper must be a MAP of (columnName,Operator).
Operator can be "=" ,"!=" ,">" ,"<>" or "<"...

```scala
DeltaHelpers.deleteFromAnotherDataframe(
  deltaTable = deltaTable,
  sourceDF = sourceDF,
  attrColNameOper = Map(("id","="),("firstname","="))
  )
```

The result of delta table will be the following:

```
+----+---------+---------+----------+
|  id|firstname| lastname|      City|
+----+---------+---------+----------+
|   2|    Maria|   Willis|    London|
|   3|     Jose| Travolta|    Mexico|
|   4|    Maria|     Pitt|    Madrid|
|   6|     Nora|   Fatehi| Marrakech|
+----+---------+---------+----------+
```


## How to contribute
We welcome contributions to this project, to contribute checkout our [CONTRIBUTING.md](CONTRIBUTING.md) file.

## How to build the project

### pre-requisites
* SBT 1.8.2
* Java 8
* Scala 2.12.12

### Building

To compile, run
`sbt compile`

To test, run
`sbt test`

To generate artifacts, run
`sbt package`

## Project maintainers

* Matthew Powers aka [MrPowers](https://github.com/MrPowers)
* Brayan Jules aka [brayanjuls](https://github.com/brayanjuls)

## More about Jodie

See [this video](https://www.youtube.com/watch?v=llHKvaV0scQ) for more info about the awesomeness of Jodie!
