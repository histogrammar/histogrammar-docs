---
title: Collecting data in Spark
type: default
toc: false
---

## Preliminaries

This tutorial uses the Scala version of Histogrammar. See the [installation guide](../../install) for installing version 0.7.1 or later.

It also uses Apache Spark. You might already have access to a Spark cluster (and that's why you're here, after all), but if you don't, you can install it yourself [from Spark's website](http://spark.apache.org/downloads.html). Spark can run on a single computer for testing, though its performance advantage comes from parallelizing across a network. The interface on a single computer is identical to the distributed version. For a single-computer installation, choose "pre-built for Hadoop 1.X" (you don't need Hadoop to be installed, only Java). Histogrammar 0.7.1 is built for a version of Scala that is compatible with Spark 1.6.1, so use that, not 2.0.

Start Spark with Histogrammar loaded like this (and combine with any other options your cluster needs; the following is sufficient for a single-computer installation):

```bash
spark-shell --jars=histogrammar-0.7.1.jar
```

Finally, this tutorial uses the CMS public data as an example. You can load it into Spark by first following the instructions on the [CMS data page](../scala-cmsdata)* and then uploading them to Spark with this command:

```scala
val rdd = sc.parallelize(events.toList)
```

Unlike the other tutorials that download events as you need them, this one downloads them all and puts them in your local Spark installation. It can take 20-30 seconds. Once that's one, you're ready to start analyzing data.

(*If you're familiar with Scala and are thinking about using `:paste` mode to enter the CMS dataset code, don't. It breaks type inference in Spark's `aggregate` method.)

## First histogram

Simple things first: you have some data in Spark (`rdd`) and you want to see how they're distributed. How do you to that in Histogrammar?

Like this:

```scala
import org.dianahep.histogrammar._

val empty = Bin(10, 0, 100, {event: Event => event.met.pt})

val filled = rdd.aggregate(empty)(new Increment, new Combine)

import org.dianahep.histogrammar.ascii._
filled.println
```

The output should look like

```
                       0                                                     133316
                       +----------------------------------------------------------+
underflow     0        |                                                          |
[  0 ,  10 )  6.697E+4 |*****************************                             |
[  10,  20 )  1.212E+5 |*****************************************************     |
[  20,  30 )  1.047E+5 |**********************************************            |
[  30,  40 )  8.037E+4 |***********************************                       |
[  40,  50 )  5.257E+4 |***********************                                   |
[  50,  60 )  2.605E+4 |***********                                               |
[  60,  70 )  1.035E+4 |*****                                                     |
[  70,  80 )  3913     |**                                                        |
[  80,  90 )  1525     |*                                                         |
[  90,  100)  717      |                                                          |
overflow      1061     |                                                          |
nanflow       0        |                                                          |
                       +----------------------------------------------------------+
```

### What just happened?

The second line,

```scala
val empty = Bin(10, 0, 100, {event: Event => event.met.pt})
```

defines an empty histogram that would count missing transverse energy (MET) in ten bins. Then instead of filling it directly, we pass it to Spark's aggregate:

```scala
val filled = rdd.aggregate(empty)(new Increment, new Combine)
```

Spark sends copies of the empty histogram to all computers in the network along with the `Increment` and `Combine` functions (created on the spot with `new`, satisfying the type constraints). The partial histograms are remotely incremented on the distributed dataset and then combined as they're gathered back. The dataset could be enormous, but the only data that get transmitted across the network are in the reduced form: the histogram. If you're using a single-computer installation, this process is only simulated.

![Scatter-gather with increment and combine](aggregate.png)

Notice that the histogram you got back is not the one you sent:

```scala
scala> println(empty.entries)
0.0

scala> println(filled.entries)
469384.0
```

Spark manages the partial copies and the merging, including clean-up and recomputation if a remote node fails. The `aggregate` line above is equivalent to creating a histogram-filling script, submitting it to a computing cluster, resubmitting failed jobs, then downloading only the good results and adding them bin-by-bin. Since it's a one-liner, it's easy enough to use in exploratory data analysis.

Finally, about the ASCII-art histogram: this tutorial is meant to highlight the aggregation method only. For a better plotting front-end, combine it with one of the other tutorials. For instance, you could plot the results directly [in Scala with the Bokeh library](../scala-spark-bokeh) or have Scala write to a pipe and view the events in a Python plotter (such as [Matplotlib](../python-basic) or [PyROOT](../python-pyroot)) using [HistogrammarWatch](../python-hgwatch).

## Many histograms at once

Transporting data is almost always more expensive than performing calculations, so it's usually advantageous to submit one job that fills a hundred histograms than a hundred jobs filling one histogram each. (Even if you use Spark's `rdd.persist()` to cache data in RAM, there's still the transport from RAM to CPU.) As you develop your work, you'll want to bind histograms together.

Here's how to do that:

```scala
val empty = Label(
  "METx" -> Bin(10, 0, 100, {event: Event => event.met.px}),
  "METy" -> Bin(10, 0, 100, {event: Event => event.met.py}),
  "MET" -> Bin(10, 0, 100, {event: Event => event.met.pt}),
  "numPrimary" -> Bin(10, 0, 20, {event: Event => event.numPrimaryVertices}))

val filled = rdd.aggregate(empty)(new Increment, new Combine)
```

Now `empty` is four empty histograms, each with a different label. If you're not familiar with Scala, the `->` syntax creates key-value pairs in the same sense as `:` in a Python dictionary. The `Label` is a member of the same class as `Bin`, with increment and combine operations, so it fills the same way. To get individual histograms out, do this:

```scala
filled("METx").println
filled("METy").println
filled("MET").println
filled("numPrimary").println
```

**[Label](../../specification/#label-directory-with-string-based-keys)** and **[Bin](../../specification/#bin-regular-binning-for-histograms)** are both interchangeable primitives. The idea is that you can build any kind of aggregation by combining primitives in different ways. For instance, if you wanted a histogram whose bins are directories of other histograms, you could do that.

```scala
val empty = Bin(5, 0, 20, {event: Event => event.numPrimaryVertices}, Label(
  "METx" -> Bin(10, 0, 100, {event: Event => event.met.px}),
  "METy" -> Bin(10, 0, 100, {event: Event => event.met.py}),
  "MET" -> Bin(10, 0, 100, {event: Event => event.met.pt})))

val filled = rdd.aggregate(empty)(new Increment, new Combine)
```

Here is the MET for number of primary vertices between 0 and 4:

```scala
println(filled.range(0))

filled.values(0)("MET").println
```
```
(0.0,4.0)

                       0                                                    11316.8
                       +----------------------------------------------------------+
underflow     0        |                                                          |
[  0 ,  10 )  7482     |**************************************                    |
[  10,  20 )  1.029E+4 |*****************************************************     |
[  20,  30 )  8859     |*********************************************             |
[  30,  40 )  9110     |***********************************************           |
[  40,  50 )  6024     |*******************************                           |
[  50,  60 )  2471     |*************                                             |
[  60,  70 )  830      |****                                                      |
[  70,  80 )  317      |**                                                        |
[  80,  90 )  109      |*                                                         |
[  90,  100)  61       |                                                          |
overflow      132      |*                                                         |
nanflow       0        |                                                          |
                       +----------------------------------------------------------+
```

And here it is between 4 and 8:

```scala
println(filled.range(1))

filled.values(1)("MET").println
```
```
(4.0,8.0)

                       0                                                    58080.0
                       +----------------------------------------------------------+
underflow     0        |                                                          |
[  0 ,  10 )  3.070E+4 |*******************************                           |
[  10,  20 )  5.280E+4 |*****************************************************     |
[  20,  30 )  4.334E+4 |*******************************************               |
[  30,  40 )  3.438E+4 |**********************************                        |
[  40,  50 )  2.345E+4 |***********************                                   |
[  50,  60 )  1.132E+4 |***********                                               |
[  60,  70 )  4216     |****                                                      |
[  70,  80 )  1505     |**                                                        |
[  80,  90 )  643      |*                                                         |
[  90,  100)  290      |                                                          |
overflow      486      |                                                          |
nanflow       0        |                                                          |
                       +----------------------------------------------------------+
```

A directory of histograms is a perfectly valid thing to use as a histogram's bin. I hope the application of this is clear: if you have a suite of histograms and your advisor (or somebody) asks for them all to be split up by number of primary vertices (or something), you can wrap a primitive and submit another Spark job immediately. Fast turn-around is key to exploration without losing focus.

### Different kinds of bundles

The "Label" primitive makes a directory of histograms, indexed by strings (names). One llimitation that may not be apparent above is that the contents all have to have the same type. They can be histograms of different quantities with different binnings, but they can't, for instance, be a few one-dimensional histograms and a few profile plots:

```scala
val hist1d = Bin(10, 0, 100, {event: Event => event.met.pt})
val profile = Bin(10, 0, 20, {event: Event => event.numPrimaryVertices},
                  Deviate({event: Event => event.met.pt}))

val fails = Label("MET" -> hist1d,
                  "MET by numPrimary" -> profile)
```

(That should give you a lot of type errors.)

One solution is to use an alternative that doesn't force them all to have the same type by not keeping track of the types of its contents:

```scala
val succeeds = UntypedLabel("MET" -> hist1d,
                            "MET by numPrimary" -> profile)

val successfullyFilled = rdd.aggregate(succeeds)(new Increment, new Combine)
```

The problem with this solution is that you can't immediately plot it. Scala has forgotten what its type was.

```scala
successfullyFilled("MET").println
```
```
<console>:61: error: value println is not a member of org.dianahep.histogrammar.Container[_$17]
              successfullyFilled("MET").println
                                        ^
```

(The `_$17` is a wildcard: it's telling you all it knows about this type.) This is _only_ an issue if you're trying to plot it in Scala. If you're converting it to JSON and sending it to a dynamically typed language like Python, it's not an issue.

```scala
println(successfullyFilled.toJson.stringify)
```
```
{"type": "UntypedLabel", "data": {"entries": 469384.0, "data": {"MET": {"type": "Bin", "data": {"low": 0.0, "high": 100.0, "entries": 469384.0, "values:type": "Count", "values": [66974.0, 121196.0, 104660.0, 80368.0, 52571.0, 26050.0, 10349.0, 3913.0, 1525.0, 717.0], "underflow:type": "Count", "underflow": 0.0, "overflow:type": "Count", "overflow": 1061.0, "nanflow:type": "Count", "nanflow": 0.0}}, "MET by numPrimary": {"type": "Bin", "data": {"low": 0.0, "high": 20.0, "entries": 469384.0, "values:type": "Deviate", "values": [{"entries": 0.0, "mean": 27.302082852852546, "variance": 313.51435166669216}, {"entries": 0.0, "mean": 27.077819903379584, "variance": 295.43035230980416}, {"entries": 0.0, "mean": 26.36247346049765, "variance": 288.9416737362195}, {"entries": 0.0, "mean": 25.61138311795298, "variance": 272.8537321773764}, {"entries": 0.0, "mean": 25.327916025016904, "variance": 265.16261897519115}, {"entries": 0.0, "mean": 25.6714575301831, "variance": 262.0688989447217}, {"entries": 0.0, "mean": 25.61728906412959, "variance": 249.06481615343583}, {"entries": 0.0, "mean": 26.63973788758004, "variance": 265.1383646086054}, {"entries": 0.0, "mean": 28.005835823800393, "variance": 410.9308920624735}, {"entries": 0.0, "mean": 29.358203252943845, "variance": 414.39235373545506}], "underflow:type": "Count", "underflow": 0.0, "overflow:type": "Count", "overflow": 1080.0, "nanflow:type": "Count", "nanflow": 0.0}}}}}
```

is a string and Python knows what to do with it.

If you _are_ working in Scala, you could try casting it (as you would in ROOT when resolving members of a `TObjArray`):

```scala
successfullyFilled("MET").asInstanceOf[hist1d.Type].println
```
```
                       0                                                     133316
                       +----------------------------------------------------------+
underflow     0        |                                                          |
[  0 ,  10 )  6.697E+4 |*****************************                             |
[  10,  20 )  1.212E+5 |*****************************************************     |
[  20,  30 )  1.047E+5 |**********************************************            |
[  30,  40 )  8.037E+4 |***********************************                       |
[  40,  50 )  5.257E+4 |***********************                                   |
[  50,  60 )  2.605E+4 |***********                                               |
[  60,  70 )  1.035E+4 |*****                                                     |
[  70,  80 )  3913     |**                                                        |
[  80,  90 )  1525     |*                                                         |
[  90,  100)  717      |                                                          |
overflow      1061     |                                                          |
nanflow       0        |                                                          |
                       +----------------------------------------------------------+
```

where `asInstanceOf` performs a cast and `hist1d.Type` is a shortcut to the one-dimensional histogram's type. Without it, we'd have to write

```scala
successfullyFilled("MET").asInstanceOf[Binning[Event, Counting, Counting, Counting, Counting]].println
```

(The `Binning` is parameterized by the kind of data it gets filled with, `Event`, and the kinds of sub-aggregators that it uses for values, underflow, overflow, and "nanflow," a count of NaN values.)

In addition to `Label` and `UntypedLabel`, which map string names to histograms (or more generally, "aggregators"), you can collect lists of aggregators without giving them names:

| Indexes are | All same type | Multiple types |
|:-|:-------------------------|:--------------|
| strings (map) | **[Label](../../specification/#label-directory-with-string-based-keys)** | **[UntypedLabel](../../specification/#untypedlabel-directory-of-different-types)** |
| integers (sequence) | **[Index](../../specification/#index-list-with-integer-keys)** | **[Branch](../../specification/#branch-tuple-of-different-types)** |

`Index` is a simple list, whose indexes range from zero until the number of items, while `Branch` attempts some Scala type programming to let each item have a different type, yet remember what those types are. Consequently, you can access items with `i0`, `i1`, etc. without casting.

This suggests an alternate to `UntypedLabel` that is less annoying in Scala: put all aggregators of one type in one branch, another type in another branch, etc. That would allow us to pull out plots without any casting.

```scala
val branch = Branch(
  Label("MET" -> Bin(10, 0, 100, {event: Event => event.met.pt})),
  Label("MET by numPrimary" ->
    Bin(10, 0, 20, {event: Event => event.numPrimaryVertices},
        Deviate({event: Event => event.met.pt}))))

val branchFilled = rdd.aggregate(branch)(new Increment, new Combine)

branchFilled.i0("MET").println
branchFilled.i1("MET by numPrimary").println
```
```
                       0                                                     133316
                       +----------------------------------------------------------+
underflow     0        |                                                          |
[  0 ,  10 )  6.697E+4 |*****************************                             |
[  10,  20 )  1.212E+5 |*****************************************************     |
[  20,  30 )  1.047E+5 |**********************************************            |
[  30,  40 )  8.037E+4 |***********************************                       |
[  40,  50 )  5.257E+4 |***********************                                   |
[  50,  60 )  2.605E+4 |***********                                               |
[  60,  70 )  1.035E+4 |*****                                                     |
[  70,  80 )  3913     |**                                                        |
[  80,  90 )  1525     |*                                                         |
[  90,  100)  717      |                                                          |
overflow      1061     |                                                          |
nanflow       0        |                                                          |
                       +----------------------------------------------------------+

                               25.7964                                      29.5048
                               +--------------------------------------------------+
[  0 ,  2 )  27.11 +-  0.3336  |             |----+---|                           |
[  2 ,  4 )  27.36 +-  0.08395 |                    |+|                           |
[  4 ,  6 )  26.93 +-  0.05595 |               +|                                 |
[  6 ,  8 )  26.68 +-  0.05110 |           |+|                                    |
[  8 ,  10)  26.89 +-  0.05600 |              |+|                                 |
[  10,  12)  27.18 +-  0.06726 |                  |+|                             |
[  12,  14)  27.32 +-  0.08648 |                   |-+|                           |
[  14,  16)  27.54 +-  0.1256  |                      |-+|                        |
[  16,  18)  28.00 +-  0.1976  |                           |--+-|                 |
[  18,  20)  28.22 +-  0.3250  |                            |----+---|            |
                               +--------------------------------------------------+
```

## Fluent Spark

All of the examples so far have presented attributes of the whole physics event: MET and number of primary vertices. This was to avoid complex Spark manipulations when the focus of the discussion was on histogramming. In this last section, I'll show some Spark idioms that can be useful in physics analyses.
