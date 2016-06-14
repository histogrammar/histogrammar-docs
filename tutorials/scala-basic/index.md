---
title: Basic use in Scala
type: default
toc: false
---

## Preliminaries

This tutorial uses the Scala version of Histogrammar. See the [installation guide](../../install) for installing version 0.7 or later.

It also uses the [CMS public dataset](../scala-cmsdata). Set up an iterator named `events` in your console. You will need a network connection. If your `events` iterator ever runs out, refresh it with

```scala
val events = EventIterator()
```

## First histogram

Simple things first: you have some data in Scala (`events`) and you want to see how they're distributed. How do you do that in Histogrammar?

Like this:

```scala
import org.dianahep.histogrammar._

val histogram = Bin(10, 0, 100, {event: Event => event.met.pt})

for (event <- events.take(1000))
  histogram.fill(event)

import org.dianahep.histogrammar.ascii._
histogram.println
```

The output should be something like

```
                  0                                                         281.600
                  +---------------------------------------------------------------+
underflow     0   |                                                               |
[  0 ,  10 )  152 |**********************************                             |
[  10,  20 )  256 |*********************************************************      |
[  20,  30 )  215 |************************************************               |
[  30,  40 )  160 |************************************                           |
[  40,  50 )  130 |*****************************                                  |
[  50,  60 )  51  |***********                                                    |
[  60,  70 )  24  |*****                                                          |
[  70,  80 )  10  |**                                                             |
[  80,  90 )  1   |                                                               |
[  90,  100)  1   |                                                               |
overflow      0   |                                                               |
nanflow       0   |                                                               |
                  +---------------------------------------------------------------+
```

### What just happened?

The first line,

```scala
import org.dianahep.histogrammar._
```

imports the basic functions and classes you need to define plots. The next gets to the heart of what Histogrammar is all about. It defines an empty histogram:

```scala
val histogram = Bin(10, 0, 100, {event: Event => event.met.pt})
```

This object aggregates a stream of input `Events`, dividing them into ten bins from `0` to `100`. The in-place function `{event: Event => event.met.pt}` defines the quantity of interest by showing how it can be extracted from the data stream.

For physicists accustomed with the HBOOK/PAW/ROOT way of doing things, this is partly familiar: we think of histograms as containers to be filled. (Other data analysis frameworks, which typically deal with datasets that fit in a laptop's memory, fill and plot a histogram in one operation.)

What may be unfamiliar is the in-place function. Typical physics analysis scripts declare ("book") empty histograms in an initialization stage and then fill them in a loop over a zillion events. In Histogrammar, we still have a fill loop, but now the physics-specific logic of what to put in the histogram is located in the initialization phase, next to the binning parameters.

This aids bookkeeping in itself (it's easier to spot an inconsistency between the binning and the fill rule when they're right next to each other), but it also makes it easier to dispatch this fill operation to a data pipeline like Apache Spark. All of the domain-specific logic is declared when the histogram is constructed, so the fill operation can be automated.

Finally, why an ASCII-art histogram? Histogrammar is not intended to replace plotting packages: its purpose is to get data into them. Plotting front-ends copy data from Histogrammar's classes into your plotting library of choice, and the other tutorials in this series cover each of these front-ends. For the sake of quick visualization, though, the Scala base package has ASCII-art methods accessible via

```scala
import org.dianahep.histogrammar.ascii._
println(histogram.ascii)   # or just
histogram.println
```

## Composability

Another fundamental aspect of Histogrammar is hidden in the above. The `Bin` constructor has several arguments with default values:

  * `value`: how to fill a bin between the low and high edges;
  * `underflow`: what to do with data below the low edge;
  * `overflow`: what to do with data above the high edge;
  * `nanflow`: what to do with data that is not a number (`NaN`).

Usually, we'd want to count the number of entries in each bin and count the underflow and overflow (and "nanflow," included so that every input value fills _some_ bin). Therefore, the default for each of these arguments is `Count()`.

To see this, try printing out `histogram` and its parts:

```scala
println(histogram)
<Binning num=10 low=0.0 high=100.0 values=Count underflow=Count overflow=Count nanflow=Count>

println(histogram.values)
List(<Counting 152.0>, <Counting 256.0>, <Counting 215.0>, <Counting 160.0>, <Counting 130.0>, <Counting 51.0>, <Counting 24.0>, <Counting 10.0>, <Counting 1.0>, <Counting 1.0>)

println(histogram.values(0).entries)
152.0
```

Binning and counting are similar things: they can both summarize a data stream. They summarize it in different ways, but they can be drop-in replacements for one another. For instance, here is a two-dimensional histogram:

```scala
val hist2d = Bin(10, 0, 100, {event: Event => event.met.px},
                 value = Bin(10, 0, 100, {event: Event => event.met.py}))

for (event <- events.take(1000))
  hist2d.fill(event)

for (i <- 0 until 10) {
  for (j <- 0 until 10)
    print("%3.0f ".format(hist2d.values(i).values(j).entries))
  println()
}
 34  24  13   6   3   2   1   0   0   0 
 36  25   6   1   2   1   0   0   0   0 
 11  11   9   1   1   1   0   0   0   0 
  8   6   5   7   4   1   0   0   0   0 
  3   0   2   2   0   0   0   0   0   0 
  0   1   0   2   0   0   0   0   0   0 
  0   0   0   0   0   0   0   0   0   0 
  0   0   0   0   0   0   0   0   0   0 
  0   0   0   0   0   0   0   0   0   0 
  0   0   0   0   0   0   0   0   0   0 
```

(Sorry, no ASCII-art for two-dimensional histograms... yet.)

In place of the first binning's counter, we used another binning. Three or four dimensional summaries of a dataset can be produced the same way, though they would be hard to visualize in any plotting library.

We also could have used a counter on its own:

```scala
val count = Count()

for (event <- events.take(1000))
  count.fill(event)

println(count)
<Counting 1000.0>
```

though it's not very enlightening.

### Plotting anything

With binning and counting, we can make histograms of any dimension, but that's still pretty limited. To plot anything, we only need a good set of aggregators and some imagination. For instance, suppose we have an aggregator that averages:

```scala
val average = Average({event: Event => event.met.pt})

for (event <- events.take(1000))
  average.fill(event)

println(average)
<Averaging mean=25.835336155264137>
```

We can combine this with binning to see the average per bin. For instance, we could compute the average pT (`event.met.pt`) per angle phi (`Math.atan2(event.met.py, event.met.px)`). In a cylindrically symmetric experiment, there should be no deviation versus phi.

```scala
val pt_vs_phi = Bin(10, 0, 2*Math.PI, {event: Event => event.met.pt},
                    value = Average({event: Event => Math.atan2(event.met.py, event.met.px)}))

val events = EventIterator()
for (event <- events.take(100000))
  pt_vs_phi.fill(event)

pt_vs_phi.println
```

The above produces

```
                            -0.180868                      0               0.138787
                            +------------------------------+----------------------+
[  0    ,  0.628) -0.1542   |    +                         |                      |
[  0.628,  1.26 ) -0.1170   |           +                  |                      |
[  1.26 ,  1.88 ) -0.1064   |            +                 |                      |
[  1.88 ,  2.51 )  0.1121   |                              |                  +   |
[  2.51 ,  3.14 ) -0.1184   |          +                   |                      |
[  3.14 ,  3.77 ) -0.04311  |                       +      |                      |
[  3.77 ,  4.40 )  0.008263 |                              |+                     |
[  4.40 ,  5.03 ) -0.009705 |                            + |                      |
[  5.03 ,  5.65 ) -0.07129  |                  +           |                      |
[  5.65 ,  6.28 ) -0.1339   |        +                     |                      |
                            +------------------------------+----------------------+
```

Oh no! That doesn't look flat versus phi&mdash; the points seem to be scattered around zero. Perhaps we need error bars.

For that, we'd need to keep track of the variance in each bin, so consider an aggregator that accumulates the variance in addition to the mean.

```scala
val deviations = Deviate({event: Event => event.met.pt})

for (event <- events.take(1000))
  deviations.fill(event)

println(deviations)
<Deviating mean=25.66900474862857, variance=293.20009419456557>
```

Now we can build a classic "profile plot," which displays the mean and the error on the mean of each bin.

```scala
val pt_vs_phi = Bin(10, 0, 2*Math.PI, {event: Event => event.met.pt},
                    value = Deviate({event: Event => Math.atan2(event.met.py, event.met.px)}))

val events = EventIterator()
for (event <- events.take(100000))
  pt_vs_phi.fill(event)

pt_vs_phi.println
```

And we see that the binwise averages are all consistent with zero.

```
                                        -0.954890                0         0.646430
                                        +------------------------+----------------+
[  0    ,  0.628) -0.1542   +-  0.2224  |               |-----+--|-|              |
[  0.628,  1.26 ) -0.1170   +-  0.1187  |                  |--+--|                |
[  1.26 ,  1.88 ) -0.1064   +-  0.09098 |                   |--+-|                |
[  1.88 ,  2.51 )  0.1121   +-  0.08037 |                        ||-+-|           |
[  2.51 ,  3.14 ) -0.1184   +-  0.06990 |                    |+-||                |
[  3.14 ,  3.77 ) -0.04311  +-  0.06337 |                      |+||               |
[  3.77 ,  4.40 )  0.008263 +-  0.05698 |                       ||+|              |
[  4.40 ,  5.03 ) -0.009705 +-  0.05476 |                       ||-|              |
[  5.03 ,  5.65 ) -0.07129  +-  0.05259 |                     |-+|                |
[  5.65 ,  6.28 ) -0.1339   +-  0.04987 |                    |+| |                |
                                        +------------------------+----------------+
```

### Alternative binning

