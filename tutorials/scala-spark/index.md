---
title: 
type: default
toc: false
---

## Preliminaries

This tutorial uses the Scala version of Histogrammar. See the [installation guide](../../install) for installing version 0.7 or later.

It also uses Apache Spark. You might already have access to a Spark cluster (and that's why you're here, after all), but if you don't, you can install it yourself [from Spark's website](http://spark.apache.org/downloads.html). Spark can run on a single computer for testing, though its performance advantage comes from parallelizing across a network. The interface on a single computer is identical to the distributed version. For a single-computer installation, choose "pre-built for Hadoop 1.X" (you don't need Hadoop to be installed, only Java). Histogrammar 0.7 is built for a version of Scala that is compatible with Spark 1.6.1, so use that, not 2.0.

Start Spark with Histogrammar loaded like this (and combine with any other options your cluster needs; the following is sufficient for a single-computer installation):

```bash
spark-shell --jars=histogrammar-0.7.jar
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

Spark sends copies of the empty histogram to all computers in the network along with the `Increment` and `Combine` functions (created on the spot with `new`). The partial histograms are remotely incremented on the distributed dataset and then combined as they're gathered back. The dataset could be enormous, but the only data that get transmitted across the network are in the reduced form: the histogram. If you're using a single-computer installation, this process is only simulated.

![Scatter-gather with increment and combine](aggregate.png)

Notice that the histogram you got back is not the one you sent:

```scala
scala> println(empty.entries)
0.0

scala> println(filled.entries)
469384.0
```

Spark manages the partial copies and the merging, including clean-up and recomputation if a remote node fails. The code above is equivalent to creating a histogram-filling script, submitting it to a computing cluster, resubmitting failed jobs, then downloading only the good results and adding them bin-by-bin. As a one-liner, it becomes easy enough to use in exploratory data analysis.

Finally, about the ASCII-art histogram: this tutorial is meant to highlight the aggregation method only, you can combine it with one of the other tutorials for a better plotting front-end. For instance, you could plot the results directly [in Scala with the Bokeh library](../scala-spark-bokeh) or have Scala write to a pipe and view the events in a Python plotter (such as [Matplotlib](../python-basic) or [PyROOT](../python-pyroot)) using [HistogrammarWatch](../python-hgwatch).

## Many histograms at once

As a batch system, Spark is optimized for large jobs. Submitting one job that fills a hundred histograms is faster than submitting a hundred jobs, filling one histogram each. Soon, you'll want to bind histograms together to submit them as a single bundle.





