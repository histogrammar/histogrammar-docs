---
title: Basic use in Scala
type: default
toc: false
summary: |
    <p>If you're using Histogrammar for the first time in <a href="http://www.scala-lang.org/">Scala</a>, read this page.</p>
    <p><b>Author:</b> <a href="http://github.com/jpivarski">Jim Pivarski</a></p>
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
val hist2d = Bin(10, -100, 100, {event: Event => event.met.px},
                 value = Bin(10, -100, 100, {event: Event => event.met.py}))

for (event <- events.take(1000))
  hist2d.fill(event)

for (i <- 0 until 10) {
  for (j <- 0 until 10)
    print("%3.0f ".format(hist2d.values(i).values(j).entries))
  println()
}
  0   0   0   0   0   0   0   0   0   0 
  0   0   1   1   0   2   0   0   1   0 
  0   0   0   3   9   9   3   0   0   0 
  0   1   9  14  36  32  18   3   0   0 
  0   2   9  48 105 106  41   7   0   1 
  0   2  11  45 139 119  26   8   1   0 
  0   2   8  21  54  36  22   7   0   0 
  0   0   2   9  14   4   6   0   0   0 
  0   0   0   0   2   0   0   0   0   0 
  0   0   0   0   1   0   0   0   0   0 
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
val pt_vs_phi = Bin(30, -Math.PI, Math.PI,
                    {event: Event => Math.atan2(event.met.py, event.met.px)},
                    value = Average({event: Event => event.met.pt}))

val events = EventIterator()
for (event <- events.take(100000))
  pt_vs_phi.fill(event)

pt_vs_phi.println
```

The above produces

```
                         24.6564                                            26.6993
                         +--------------------------------------------------------+
[ -3.14 , -2.93 )  25.15 |              +                                         |
[ -2.93 , -2.72 )  26.26 |                                            +           |
[ -2.72 , -2.51 )  25.24 |                +                                       |
[ -2.51 , -2.30 )  25.75 |                              +                         |
[ -2.30 , -2.09 )  25.85 |                                 +                      |
[ -2.09 , -1.88 )  26.07 |                                       +                |
[ -1.88 , -1.68 )  26.02 |                                      +                 |
[ -1.68 , -1.47 )  25.50 |                       +                                |
[ -1.47 , -1.26 )  25.89 |                                  +                     |
[ -1.26 , -1.05 )  26.53 |                                                   +    |
[ -1.05 , -0.838)  26.04 |                                      +                 |
[ -0.838, -0.628)  25.79 |                               +                        |
[ -0.628, -0.419)  26.18 |                                          +             |
[ -0.419, -0.209)  25.27 |                 +                                      |
[ -0.209,  0    )  26.21 |                                           +            |
[  0    ,  0.209)  25.92 |                                   +                    |
[  0.209,  0.419)  25.19 |               +                                        |
[  0.419,  0.628)  25.76 |                              +                         |
[  0.628,  0.838)  25.57 |                         +                              |
[  0.838,  1.05 )  25.80 |                               +                        |
[  1.05 ,  1.26 )  25.45 |                      +                                 |
[  1.26 ,  1.47 )  24.83 |     +                                                  |
[  1.47 ,  1.68 )  25.51 |                       +                                |
[  1.68 ,  1.88 )  25.10 |            +                                           |
[  1.88 ,  2.09 )  24.91 |       +                                                |
[  2.09 ,  2.30 )  25.08 |            +                                           |
[  2.30 ,  2.51 )  25.50 |                       +                                |
[  2.51 ,  2.72 )  25.15 |              +                                         |
[  2.72 ,  2.93 )  25.04 |          +                                             |
[  2.93 ,  3.14 )  25.47 |                      +                                 |
                         +--------------------------------------------------------+
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
val pt_vs_phi = Bin(30, -Math.PI, Math.PI,
                    {event: Event => Math.atan2(event.met.py, event.met.px)},
                    value = Deviate({event: Event => event.met.pt}))

val events = EventIterator()
for (event <- events.take(100000))
  pt_vs_phi.fill(event)

pt_vs_phi.println
```

And we see that the binwise averages are all consistent with each other.

```
                                    23.5800                                 27.6424
                                    +---------------------------------------------+
[ -3.14 , -2.93 )  25.15 +-  0.2889 |              |--+---|                       |
[ -2.93 , -2.72 )  26.26 +-  0.3171 |                          |---+--|           |
[ -2.72 , -2.51 )  25.24 +-  0.2857 |               |--+---|                      |
[ -2.51 , -2.30 )  25.75 +-  0.2849 |                     |--+--|                 |
[ -2.30 , -2.09 )  25.85 +-  0.2794 |                      |--+--|                |
[ -2.09 , -1.88 )  26.07 +-  0.2715 |                         |--+--|             |
[ -1.88 , -1.68 )  26.02 +-  0.2674 |                        |--+--|              |
[ -1.68 , -1.47 )  25.50 +-  0.2515 |                  |--+--|                    |
[ -1.47 , -1.26 )  25.89 +-  0.2618 |                       |--+--|               |
[ -1.26 , -1.05 )  26.53 +-  0.2583 |                              |--+--|        |
[ -1.05 , -0.838)  26.04 +-  0.2594 |                        |--+--|              |
[ -0.838, -0.628)  25.79 +-  0.2565 |                      |-+--|                 |
[ -0.628, -0.419)  26.18 +-  0.2585 |                          |--+--|            |
[ -0.419, -0.209)  25.27 +-  0.2620 |                |--+--|                      |
[ -0.209,  0    )  26.21 +-  0.2851 |                          |--+--|            |
[  0    ,  0.209)  25.92 +-  0.2750 |                       |--+--|               |
[  0.209,  0.419)  25.19 +-  0.2913 |               |--+--|                       |
[  0.419,  0.628)  25.76 +-  0.2874 |                     |--+--|                 |
[  0.628,  0.838)  25.57 +-  0.2855 |                   |--+--|                   |
[  0.838,  1.05 )  25.80 +-  0.3084 |                     |---+--|                |
[  1.05 ,  1.26 )  25.45 +-  0.3011 |                 |---+--|                    |
[  1.26 ,  1.47 )  24.83 +-  0.3027 |          |---+--|                           |
[  1.47 ,  1.68 )  25.51 +-  0.3066 |                  |--+---|                   |
[  1.68 ,  1.88 )  25.10 +-  0.3112 |             |---+--|                        |
[  1.88 ,  2.09 )  24.91 +-  0.3088 |           |---+--|                          |
[  2.09 ,  2.30 )  25.08 +-  0.3047 |             |---+--|                        |
[  2.30 ,  2.51 )  25.50 +-  0.3130 |                  |--+---|                   |
[  2.51 ,  2.72 )  25.15 +-  0.3249 |              |--+---|                       |
[  2.72 ,  2.93 )  25.04 +-  0.3053 |             |--+---|                        |
[  2.93 ,  3.14 )  25.47 +-  0.3154 |                 |---+--|                    |
                                    +---------------------------------------------+
```

### Alternative binning

So far, we have only replaced the counters in a regularly binned histogram with different functions. We should be able to replace the binning procedure itself.

A regular histogram is like a dense vector of aggregators. Every bin is allocated and initially populated with zero. We could also consider the sparse case, in which bins are only allocated when they are not filled with zero.

To allocate a sparsely binned histogram, we don't need to specify the low and high edges, just the bin width (`10.0` here). Any finite integer is a valid bin index (in practice limited to 64-bit signed integers). We do, however, need to specify a bin width. This is useful for cases in which little is known about the distribution of interest: only a characteristic scale.

```scala
val sparse = SparselyBin(10.0, {event: Event => event.met.px})

for (event <- events.take(1000))
  sparse.fill(event)

println(sparse.bins)
Map(8 -> <Counting 1.0>, -1 -> <Counting 198.0>, -7 -> <Counting 3.0>, 2 -> <Counting 95.0>, -4 -> <Counting 39.0>, 5 -> <Counting 12.0>, 4 -> <Counting 23.0>, -5 -> <Counting 16.0>, 7 -> <Counting 1.0>, -2 -> <Counting 121.0>, 1 -> <Counting 150.0>, -8 -> <Counting 2.0>, 3 -> <Counting 55.0>, -6 -> <Counting 8.0>, 6 -> <Counting 1.0>, -3 -> <Counting 74.0>, 0 -> <Counting 201.0>)
```

Histogrammar represents this kind of aggregator as a hashmap from bin indexes to counters. Needless to say, we could have filled it with averages or deviations or other binning methods.

Here is a sparse two-dimensional histogram. Drawing it is tricky because one has to evade the unfilled indexes.

```scala
val hist2d = SparselyBin(10.0, {event: Event => event.met.px},
                 value = SparselyBin(10.0, {event: Event => event.met.py}))

val events = EventIterator()
for (event <- events.take(1000))
  hist2d.fill(event)

val min = hist2d.values.flatMap(_.minBin).min
for (i <- hist2d.minBin.get to hist2d.maxBin.get) {
  val col = hist2d.bins.get(i)
  col.flatMap(_.maxBin) match {
    case Some(max) =>
      for (j <- min to max)
        hist2d.bins.get(i).flatMap(_.bins.get(j)) match {
          case Some(counting) =>
            print("%3.0f ".format(counting.entries))
          case None =>
            print("    ")
        }
    case None =>
      print("    ")
  }
  println()
}
                          2                   1 
                                      1       1 
                  1   1   3   1   1   1               1 
      1       1   2   5   2   5   4   3   1   3   1 
          1   2   3   2   8  10   8   4   5   2   1 
      1       2   7   9   7  12   6   8   9   4 
      1   2   4   3  12  19  24  21  15  11   6   5       1 
      1   4   7   7  23  27  34  47  19  16  16   6 
      1   2   5   8  20  30  45  56  17  10   5   2   1   1 
      1   1   4   6   8  22  36  28  11   7   5   4       1 
      1           7  14  17  18  13   8   6   8   2   2 
  1       1   2   2   5  10   6   5   7   1   3 
          1   1       1       5   8   5   4   1   1 
              1   1           1   1   3 
                          2   3 
              1 
```

### Superstructures

If counting, averaging, and deviating are aggregators that typically go inside of some kind of binning, there are some aggregators that typically surround a binning. One of these computes fractions.

For decades, physicists have constructed efficiency plots manually: by booking two histograms with exactly the same binning, applying a selection to one and not the other, and then dividing them bin-by-bin. If the bin edges of the two histograms are not correctly aligned or if a code revision is applied to one and not the other, the result silently becomes meaningless.

Histogrammar makes common techniques like fractions into first-class citizens. `Fraction` is an aggregator on the same level as `Bin` and `Count`, and may be mixed freely with them.

```scala
val frac = Fraction({event: Event => event.numPrimaryVertices > 5},
                    Bin(30, 0, 100, {event: Event => event.met.pt}))

val events = EventIterator()
for (event <- events.take(1000000))
  frac.fill(event)

frac.println
```

The final plot is the fraction of events with `numPrimaryVertices > 5` as a function of `met.pt`.

```
                        0.631723                                           0.764523
                        +---------------------------------------------------------+
underflow        nan    |                                                         |
[  0   ,  3.33)  0.6432 |     +                                                   |
[  3.33,  6.67)  0.6649 |              +                                          |
[  6.67,  10  )  0.6803 |                     +                                   |
[  10  ,  13.3)  0.6995 |                             +                           |
[  13.3,  16.7)  0.7205 |                                      +                  |
[  16.7,  20  )  0.7345 |                                            +            |
[  20  ,  23.3)  0.7439 |                                                +        |
[  23.3,  26.7)  0.7303 |                                          +              |
[  26.7,  30  )  0.7139 |                                   +                     |
[  30  ,  33.3)  0.6983 |                             +                           |
[  33.3,  36.7)  0.6791 |                    +                                    |
[  36.7,  40  )  0.6665 |               +                                         |
[  40  ,  43.3)  0.6597 |            +                                            |
[  43.3,  46.7)  0.6708 |                 +                                       |
[  46.7,  50  )  0.6826 |                      +                                  |
[  50  ,  53.3)  0.6886 |                        +                                |
[  53.3,  56.7)  0.7105 |                                  +                      |
[  56.7,  60  )  0.7318 |                                           +             |
[  60  ,  63.3)  0.7308 |                                           +             |
[  63.3,  66.7)  0.7259 |                                        +                |
[  66.7,  70  )  0.7498 |                                                   +     |
[  70  ,  73.3)  0.7535 |                                                    +    |
[  73.3,  76.7)  0.7482 |                                                  +      |
[  76.7,  80  )  0.7410 |                                               +         |
[  80  ,  83.3)  0.7186 |                                     +                   |
[  83.3,  86.7)  0.7510 |                                                   +     |
[  86.7,  90  )  0.7139 |                                   +                     |
[  90  ,  93.3)  0.7117 |                                  +                      |
[  93.3,  96.7)  0.7532 |                                                    +    |
[  96.7,  100 )  0.6634 |              +                                          |
overflow         0.6428 |     +                                                   |
nanflow          nan    |                                                         |
                        +---------------------------------------------------------+
```

Typically, these plots come with error bars, but Histogrammar never assumes statistical techniques: that's the data analyst's job. A naive way to compute error bars on a fraction of counts is to treat them as binomials (valid far from zero or one).

```scala
def naiveConfidenceInterval(numer: Double, denom: Double, z: Double) = {
  val p = numer/denom
  val err = Math.sqrt(p * (1.0 - p) / denom)
  p + z*err
}

frac.println(naiveConfidenceInterval, width=80)
```

We are beginning to learn interesting things about this dataset.

```
                        0.612726                                           0.796966
                        +---------------------------------------------------------+
underflow        nan    |                                                         |
[  0   ,  3.33)  0.6432 |        |+-|                                             |
[  3.33,  6.67)  0.6649 |               |+|                                       |
[  6.67,  10  )  0.6803 |                    |+|                                  |
[  10  ,  13.3)  0.6995 |                          |+|                            |
[  13.3,  16.7)  0.7205 |                                 +|                      |
[  16.7,  20  )  0.7345 |                                     |+                  |
[  20  ,  23.3)  0.7439 |                                        |+               |
[  23.3,  26.7)  0.7303 |                                    +|                   |
[  26.7,  30  )  0.7139 |                               +|                        |
[  30  ,  33.3)  0.6983 |                          +|                             |
[  33.3,  36.7)  0.6791 |                    |+                                   |
[  36.7,  40  )  0.6665 |                |+|                                      |
[  40  ,  43.3)  0.6597 |              |+|                                        |
[  43.3,  46.7)  0.6708 |                 |+|                                     |
[  46.7,  50  )  0.6826 |                    |-+|                                 |
[  50  ,  53.3)  0.6886 |                      |+-|                               |
[  53.3,  56.7)  0.7105 |                             |+-|                        |
[  56.7,  60  )  0.7318 |                                   |-+-|                 |
[  60  ,  63.3)  0.7308 |                                   |-+-|                 |
[  63.3,  66.7)  0.7259 |                                 |-+-|                   |
[  66.7,  70  )  0.7498 |                                        |-+--|           |
[  70  ,  73.3)  0.7535 |                                        |---+--|         |
[  73.3,  76.7)  0.7482 |                                      |---+---|          |
[  76.7,  80  )  0.7410 |                                   |----+---|            |
[  80  ,  83.3)  0.7186 |                           |-----+----|                  |
[  83.3,  86.7)  0.7510 |                                     |-----+-----|       |
[  86.7,  90  )  0.7139 |                        |------+-------|                 |
[  90  ,  93.3)  0.7117 |                      |--------+-------|                 |
[  93.3,  96.7)  0.7532 |                                   |-------+--------|    |
[  96.7,  100 )  0.6634 |     |----------+---------|                              |
overflow         0.6428 |     |---+----|                                          |
nanflow          nan    |                                                         |
                        +---------------------------------------------------------+
```

### Directories

Another important kind of superstructure is the directory. Most analyses require lots of histograms, and it helps to be able to collect them into a bundle. `Label` is an aggregator that names histograms by putting them in a hashmap.

```scala
val bundle = Label(
  "MET px" -> Bin(10, 0, 100, {event: Event => event.met.px}),
  "MET py" -> Bin(10, 0, 100, {event: Event => event.met.py}),
  "num primary" -> Bin(10, 0, 100, {event: Event => event.numPrimaryVertices}))
```

Like any other aggregator, it has a `fill` method. For a collection, this method fills all of its contents.

```scala
for (event <- events.take(1000))
  bundle.fill(event)

println(bundle("MET px").entries)
1000.0

println(bundle("MET py").entries)
1000.0

println(bundle("num primary").entries)
1000.0
```

As a by-product of composability, we get subdirectories for free:

```scala
val bundle = Label(
  "MET" -> Label(
    "px" -> Bin(10, 0, 100, {event: Event => event.met.px}),
    "py" -> Bin(10, 0, 100, {event: Event => event.met.py})
  ),
  "primary" -> Label(
    "num" -> Bin(10, 0, 100, {event: Event => event.numPrimaryVertices})
  )
)

for (event <- events.take(1000))
  bundle.fill(event)

println(bundle("MET")("px").entries)
1000.0

println(bundle("MET")("py").entries)
1000.0

println(bundle("primary")("num").entries)
1000.0
```

## Interoperability

Each of these aggregators has a stable JSON representation that is validated across the different language versions of Histogrammar. That is, you can dump a histogram or group of histograms from Scala to JSON and load that JSON into equivalent structures in Python.

```scala
println(bundle.toJson.stringify)
```

produces

```json
{
  "type": "Label",
  "data": {
    "entries": 1000,
    "type": "Label",
    "data": {
      "MET": {
        "entries": 1000,
        "type": "Bin",
        "data": {
          "px": {
            "low": 0,
            "high": 100,
            "entries": 1000,
            "values:type": "Count",
            "values": [221, 172, 71, 41, 16, 12, 2, 0, 1, 0],
            "underflow:type": "Count", "underflow": 464,
            "overflow:type": "Count", "overflow": 0,
            "nanflow:type": "Count", "nanflow": 0},
          "py": {
            "low": 0,
            "high": 100,
            "entries": 1000,
            "values:type": "Count",
            "values": [198, 120, 65, 46, 27, 3, 1, 2, 0, 0],
            "underflow:type": "Count", "underflow": 538,
            "overflow:type": "Count", "overflow": 0,
            "nanflow:type": "Count", "nanflow": 0}
        }
      },
      "primary": {
        "entries": 1000,
        "type": "Bin",
        "data": {
          "num": {
            "low": 0,
            "high": 100,
            "entries": 1000,
            "values:type": "Count",
            "values": [916, 84, 0, 0, 0, 0, 0, 0, 0, 0],
            "underflow:type": "Count", "underflow": 0,
            "overflow:type": "Count", "overflow": 0,
            "nanflow:type": "Count", "nanflow": 0}
        }
      }
    }
  }
}
```

This allows you to decouple parts of your analysis. It may be easier or faster to aggregate data in one language or framework and then plot it in another. The desire to use a particular library for final plots shouldn't force you to do all of your analysis in that library.

### The grammar of Histogrammar

You may have noticed that all of the aggregator constructors are imperative verbs: `Count`, `Average`, `Bin`, `Label` while the objects themselves are represented as gerunds: `Counting`, `Averaging`, `Binning`, `Labeling`. That distinction has to do with JSON: aggregators have different features before and after serialization.

| Before serialization | After serialization |
|:---------------------|:--------------------|
| Has an associated function that determines how to fill. | Has no such function; might have been filled in another language. (If the function was named, its name is carried over to help with bookkeeping.) |
| Can be filled or combined with other aggregators of the same type. | Can only be combined with other aggregators of the same type. |
| Mutable: `entries` and fill values can change. | Immutable: `entries` and fill values are fixed. |

In statically typed languages like Scala, aggregators are named with a gerund before serialization and past-tense after serialization (`Counted`, `Averaged`, `Binned`, `Labeled`).

```scala
println(bundle)
<Labeling values=Label size=2>

val bundled = Factory.fromJson(bundle.toJson.stringify)

println(bundled)
<Labeled values=Label size=2>
```

In dynamically typed languages like Python, this distinction is not helpful: an aggregator either has an associated function or it does not, and it is always mutable. In these languages, the imperative tense is always used (`Count`, `Average`, `Bin`, `Label`).

## Fistful of aggregators

[The specification](../../specification) lists all of the available aggregators and how to use each one. However, that may be too much information for a start.

The most useful aggregators are the following. Tinker with them to get familiar; building up an analysis is easier when you know "there's an app for that."

**Simple counters:**

  * [`Count`](../../specification/#count-sum-of-weights): just counts. Every aggregator has an `entries` field, but `Count` _only_ has this field.
  * [`Average`](../../specification/#average-mean-of-a-quantity) and [`Deviate`](../../specification/#deviate-mean-and-variance): add mean and variance, cumulatively.
  * [`Minimize`](../../specification/#minimize-minimum-value) and [`Maximize`](../../specification/#maximize-maximum-value): lowest and highest value seen.

**Histogram-like objects:**

  * [`Bin`](../../specification/#bin-regular-binning-for-histograms) and [`SparselyBin`](../../specification/#sparselybin-ignore-zeros): split a numerical domain into uniform bins and redirect aggregation into those bins.
  * [`Categorize`](../../specification/#categorize-string-valued-bins-bar-charts): split a string-valued domain by unique values; good for making bar charts (which are histograms with a string-valued axis).
  * [`CentrallyBin`](#centrallybin-fully-partitioning-with-centers) and [`IrregularlyBin`](../../specification/#irregularlybin-fully-partitioning-with-edges): split a numerical domain into arbitrary subintervals, usually for separate plots like particle pseudorapidity or collision centrality.

**Collections:**

  * [`Label`](../../specification/#label-directory-with-string-based-keys), [`UntypedLabel`](../../specification/#untypedlabel-directory-of-different-types), and [`Index`](../../specification/#index-list-with-integer-keys): bundle objects with string-based keys (`Label` and `UntypedLabel`) or simply an ordered array (effectively, integer-based keys) consisting of a single type (`Label` and `Index`) or any types (`UntypedLabel`).
  * [`Branch`](../../specification/#branch-tuple-of-different-types): for the fourth case, an ordered array of any types. A `Branch` is useful as a "cable splitter". For instance, to make a histogram that tracks minimum and maximum value, do this:

```scala
val rangeHistogram = Bin(30, 0, 100, {event: Event => event.met.pt},
             Branch(Minimize({event: Event => event.numPrimaryVertices}),
                    Maximize({event: Event => event.numPrimaryVertices})))

for (event <- events.take(1000))
  rangeHistogram.fill(event)

println(rangeHistogram.values(10).i0.min)
1.0

println(rangeHistogram.values(10).i1.max)
13.0

println(rangeHistogram.values map {v => (v.i0.min, v.i1.max)})
List((0.0,16.0), (1.0,12.0), (1.0,11.0), (1.0,12.0), (1.0,12.0), (1.0,13.0), (1.0,12.0), (1.0,12.0), (1.0,13.0), (1.0,14.0), (1.0,13.0), (2.0,12.0), (1.0,10.0), (1.0,14.0), (2.0,10.0), (0.0,9.0), (3.0,10.0), (3.0,12.0), (4.0,10.0), (3.0,13.0), (3.0,9.0), (5.0,5.0), (3.0,6.0), (4.0,4.0), (8.0,8.0), (NaN,NaN), (5.0,5.0), (NaN,NaN), (NaN,NaN), (NaN,NaN))
```

**Filtering:**

  * [`Select`](../../specification/#select-apply-a-cut): general purpose cutting. If you need a histogram or a collection of histograms to be filtered by some quantity, wrap them in a `Select`.

**Non-aggregation:**

  * [`Bag`](../../specification/#bag-accumulate-values-for-scatter-plots): collects data points, rather than aggregating a quantity.
