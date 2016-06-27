---
title: Collecting data in SparkSQL
type: default
toc: false
summary: |
    <p>If you're working with <a href="http://spark.apache.org/sql/">SQL in Apache Spark</a> in Scala, read this page.</p>
    <p><b>Author:</b> <a href="http://github.com/jpivarski">Jim Pivarski</a></p>
---

## Preliminaries

This tutorial should be regarded as a sequel to [Collecting data in Spark](../scala-spark), so set up your environment in the same way with the addition of the `histogrammar-sparksql` JAR.

That is, start a Spark Shell with

```bash
spark-shell --jars=histogrammar-0.7.1.jar,histogrammar-sparksql-0.7.1.jar
```

and load the [CMS public data](../scala-cmsdata). Then make an RDD out of `events`:

```scala
val rdd = sc.parallelize(events.toList)
```

As before, you'll need to wait 10-20 seconds for the dataset to download.

## Why SparkSQL?

If you're reading these in order, you just finished a whole tutorial on plotting data in Spark. Why SparkSQL?

Plain Spark provides an "object oriented view" of your data. Events are complex, structured objects. SQL is a widely used abstraction that views data as flat tables: the columns are named and have simple types (e.g. boolean, number, string), and the rows form an unordered, unstructured collection. Business analysts call this a "relational view," and physicists call it an "ntuple."

These are soft terms, not absolutes: you can turn the object-oriented RDD into a relational SQL table. The lines below perform the conversion and print the DataFrame's schema (names and types of the columns).

```scala
val df = rdd.toDF
df.printSchema
```
```
root
 |-- jets: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- px: double (nullable = false)
 |    |    |-- py: double (nullable = false)
 |    |    |-- pz: double (nullable = false)
 |    |    |-- E: double (nullable = false)
 |    |    |-- btag: double (nullable = false)
 |-- muons: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- px: double (nullable = false)
 |    |    |-- py: double (nullable = false)
 |    |    |-- pz: double (nullable = false)
 |    |    |-- E: double (nullable = false)
 |    |    |-- q: integer (nullable = false)
 |    |    |-- iso: double (nullable = false)
 |-- electrons: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- px: double (nullable = false)
 |    |    |-- py: double (nullable = false)
 |    |    |-- pz: double (nullable = false)
 |    |    |-- E: double (nullable = false)
 |    |    |-- q: integer (nullable = false)
 |    |    |-- iso: double (nullable = false)
 |-- photons: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- px: double (nullable = false)
 |    |    |-- py: double (nullable = false)
 |    |    |-- pz: double (nullable = false)
 |    |    |-- E: double (nullable = false)
 |    |    |-- iso: double (nullable = false)
 |-- met: struct (nullable = true)
 |    |-- px: double (nullable = false)
 |    |-- py: double (nullable = false)
 |-- numPrimaryVertices: long (nullable = false)
```

As you can see, this table is not flat: the columns `jets`, `muons`, `electrons`, and `photons` are all `arrays` containing `struct` structures. While it's possible to put structured data in an SQL table, it's not convenient. Most of the SQL operations are designed for simple types.

For a better example, let's go back to `NtupleVariables` from the [Collecting data in Spark](../scala-spark) tutorial:

```scala
def cuts(event: Event): Boolean = {
  if (event.muons.size >= 2) {
    val decreasingPt = event.muons.sortBy(-_.pt)   // shorthand function definition
    val mu1 = decreasingPt(0)
    val mu2 = decreasingPt(1)
    (mu1 + mu2).mass > 60      // return true iff mass > 60
  }
  else
    false
}

case class Dimuon(mass: Double, pt: Double, eta: Double, phi: Double)

case class NtupleVariables(mu1: Muon, mu2: Muon, pair: Dimuon, numJets: Int)

def ntuple(event: Event): NtupleVariables = {
    val decreasingPt = event.muons.sortBy(-_.pt)
    val mu1 = decreasingPt(0)
    val mu2 = decreasingPt(1)
    val pair = mu1 + mu2
    NtupleVariables(mu1, mu2, Dimuon(pair.mass, pair.pt, pair.eta, pair.phi), event.jets.size)
}

val ntupleRDD = rdd.filter(cuts).map(ntuple)
```

This makes a better DataFrame:

```scala
val df = ntupleRDD.toDF
df.printSchema
```
```
 |-- mu1: struct (nullable = true)
 |    |-- px: double (nullable = false)
 |    |-- py: double (nullable = false)
 |    |-- pz: double (nullable = false)
 |    |-- E: double (nullable = false)
 |    |-- q: integer (nullable = false)
 |    |-- iso: double (nullable = false)
 |-- mu2: struct (nullable = true)
 |    |-- px: double (nullable = false)
 |    |-- py: double (nullable = false)
 |    |-- pz: double (nullable = false)
 |    |-- E: double (nullable = false)
 |    |-- q: integer (nullable = false)
 |    |-- iso: double (nullable = false)
 |-- pair: struct (nullable = true)
 |    |-- mass: double (nullable = false)
 |    |-- pt: double (nullable = false)
 |    |-- eta: double (nullable = false)
 |    |-- phi: double (nullable = false)
 |-- numJets: integer (nullable = false)
```

It has some `structs`, but `structs` that aren't in `arrays` are equivalent to naming the inner fields with dots: `mu1.px` and `mu2.px`. It's more like a namespace than a data structure.

To look at the data, type `df.show`:

```
+--------------------+--------------------+--------------------+-------+
|                 mu1|                 mu2|                pair|numJets|
+--------------------+--------------------+--------------------+-------+
|[6.99902391433715...|[-3.3582911491394...|[88.9468251097268...|      0|
|[38.4369850158691...|[-34.803283691406...|[89.8866777537253...|      0|
|[-30.284072875976...|[19.5514678955078...|[92.5088957017361...|      0|
|[-25.272924423217...|[19.8373622894287...|[87.3315168475287...|      0|
|[-4.7350287437438...|[4.49474239349365...|[86.7358985788396...|      0|
|[26.2672595977783...|[-22.861051559448...|[90.5940564776797...|      0|
|[29.9819164276123...|[-33.361652374267...|[99.5498003142394...|      0|
|[29.6038303375244...|[-22.493833541870...|[90.1605714355068...|      0|
|[-1.7054762840270...|[-19.805889129638...|[93.4717349549817...|      0|
|[7.08884572982788...|[-0.7443042993545...|[79.1500134984206...|      0|
|[32.0814361572265...|[-30.429578781127...|[84.0507769377059...|      0|
|[7.74842691421508...|[-0.3794993162155...|[93.4596587717993...|      0|
|[50.356201171875,...|[-2.1814038753509...|[91.3169886315232...|      0|
|[-24.259384155273...|[23.6800918579101...|[91.1989769010198...|      0|
|[38.6725082397460...|[-38.284599304199...|[93.8644833511309...|      0|
|[-26.027690887451...|[22.6197395324707...|[86.6052503821990...|      0|
|[24.1641025543212...|[-24.631780624389...|[91.1063914854433...|      0|
|[26.4673213958740...|[-23.045598983764...|[89.5466850476314...|      0|
|[5.78568506240844...|[2.45254373550415...|[110.406442076157...|      0|
|[-13.020010948181...|[-2.4101591110229...|[90.7228064095838...|      1|
+--------------------+--------------------+--------------------+-------+
only showing top 20 rows
```

You can filter with `where` and apply transformations with `select`. To pull out column names (including dots for `structs` and brackets for `arrays`) use SparkSQL's `$"colname"` syntax.

```scala
df.where($"numJets" >= 2).select($"pair.mass", $"pair.eta").show
```
```
+-----------------+--------------------+
|             mass|                 eta|
+-----------------+--------------------+
|92.85943239264815|  2.1419080250609013|
| 90.3544996155023|  1.8372168020957644|
|94.07325607528915|-0.28704091263097115|
|  88.893812677243|-0.04311684849064149|
|87.01650174498104|   2.365874902693869|
|90.70379486284489| -0.2186559616668296|
|91.13288235912091| -1.0201887121491608|
|90.36272436231928|  -2.792326666101461|
| 66.6879869260017|   -1.14636250544246|
|93.80370186697948|-0.02144974885299...|
|93.78736361472421|  2.1451971921732382|
|85.79339370799705|   -4.48316612241891|
|89.41774515709061|   1.892365369276679|
|91.13198920625813| 0.16942049542685106|
|91.27485498383598| -0.4113073595256221|
|86.89024507978654|-0.37054343945781565|
|91.99214911201582|  2.2050734715109113|
|93.36418538113911|  1.7091015898844444|
|92.06548152016934| -0.5675923068607271|
|91.13548921615673|  1.2278600866581004|
+-----------------+--------------------+
only showing top 20 rows
```

To make plots, use the same syntax in place of inline functions:

```scala
import org.dianahep.histogrammar._
import org.dianahep.histogrammar.ascii_
import org.dianahep.histogrammar.sparksql._

df.where($"numJets" >= 2).histogrammar(Bin(30, 60, 120, $"pair.mass")).println
```
```
                   0                                                        168.300
                   +--------------------------------------------------------------+
underflow      0   |                                                              |
[  60 ,  62 )  1   |                                                              |
[  62 ,  64 )  5   |**                                                            |
[  64 ,  66 )  5   |**                                                            |
[  66 ,  68 )  3   |*                                                             |
[  68 ,  70 )  6   |**                                                            |
[  70 ,  72 )  2   |*                                                             |
[  72 ,  74 )  3   |*                                                             |
[  74 ,  76 )  4   |*                                                             |
[  76 ,  78 )  3   |*                                                             |
[  78 ,  80 )  5   |**                                                            |
[  80 ,  82 )  9   |***                                                           |
[  82 ,  84 )  12  |****                                                          |
[  84 ,  86 )  14  |*****                                                         |
[  86 ,  88 )  33  |************                                                  |
[  88 ,  90 )  107 |***************************************                       |
[  90 ,  92 )  153 |********************************************************      |
[  92 ,  94 )  87  |********************************                              |
[  94 ,  96 )  28  |**********                                                    |
[  96 ,  98 )  10  |****                                                          |
[  98 ,  100)  5   |**                                                            |
[  100,  102)  3   |*                                                             |
[  102,  104)  4   |*                                                             |
[  104,  106)  3   |*                                                             |
[  106,  108)  0   |                                                              |
[  108,  110)  0   |                                                              |
[  110,  112)  2   |*                                                             |
[  112,  114)  1   |                                                              |
[  114,  116)  1   |                                                              |
[  116,  118)  2   |*                                                             |
[  118,  120)  2   |*                                                             |
overflow       29  |***********                                                   |
nanflow        0   |                                                              |
                   +--------------------------------------------------------------+
```

The `org.dianahep.histogrammar.sparksql._` import allows Histogrammar primitives to recognize SparkSQL columns in the `$"colname"` syntax as though they were functions. Tha's why

```scala
Bin(30, 60, 120, $"pair.mass")
```

is not an error. It also adds a `histogrammar` method to any SQL DataFrame, so that we can pass histograms to a DataFrame more conveniently than turning it into an RDD and using `aggregate`.

## Advantages and disadvantages

SQL DataFrames are especially useful when you have a lot of columns or are making deep cuts. Unlike generic Scala functions, Spark can parse expressions made out of columns and apply filters upstream, reducing the number of rows and columns that have to be loaded to be transformed. SQL queries must be fast for their intended use-case: interactive explorations of the data.

However, complex data structures and complex manipulations are harder to express in the SQL language. It is _possible_ to do manipulations on deeply structured objects like `Event`, but not easy (you start "exploding tables into lateral views" and such). Moreover, the columns can only be combined with functions that SparkSQL knows about, such as `+`, `-`, `*`, `/`, and `sqrt`. The versions of these functions overloaded for SQL make expression trees like a symbolic algebra system (Mathematica/Maple) for Spark to simplify.

Therefore, this "relational view" of your data is best suited to the end stages of your workflow, after the data have been destructrured and you need to do some exploring to determine your choice of cuts. Histogrammar can help here by supplementing SparkSQL's `show` method. It shows you what the entire dataset looks like as a plot, rather than a table of the first 20 rows.

Spark is in the process of unifying the two views into [something it calls a Dataset](https://databricks.com/blog/2016/01/04/introducing-apache-spark-datasets.html). With a Dataset, you should be able to use both methods with Histogrammar: the `$"colname"` syntax and ordinary Scala functions.
