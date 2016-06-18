---
title: Making PyROOT plots
type: default
toc: false
---

## Preliminaries

This tutorial uses the Python version of Histogrammar. See the [installation guide](../../install) for installing version 0.7 or later.

It also uses the [CMS public dataset](../python-cmsdata). Set up an iterator named `events` in your console. You will need a network connection. If your `events` iterator ever runs out, refresh it with

```scala
events = EventIterator()
```

## First histogram

Simple things first: you have some data in Python (`events`) and you want to see how they're distributed. How do you do that in Histogrammar?

Like this:

```python
from histogrammar import *

histogram = Bin(100, 0, 100, lambda event: event.met.pt)

for i, event in enumerate(events):
    if i == 1000: break
    histogram.fill(event)

roothist = histogram.root("name", "title")
roothist.Draw()
```

The output should be something like

![First plot](first.png)

### What just happened?

The first line,

```python
from histogrammar import *
```

imports the basic functions and classes you need to define plots. The next gets to the heart of what Histogrammar is all about. It defines an empty histogram:

```python
histogram = Bin(100, 0, 100, lambda event: event.met.pt)
```

As a ROOT user, you are no doubt familiar with the concept of a histogram as an empty container that must be filled to be meaningful. However, the fill rule `lambda event: event.met.pt` may be unexpected.

In a ROOT typical script, you would declare ("book") a suite of empty histograms in an initialization stage and then fill them in a loop over a zillion events. The physics-specific logic of what to put in each histogram can end up being far from the booking code.

In Histogrammar, the rule for how to fill the histogram is provided in the constructor, so that the loop can be automated with no input from the data analyst. Although this is useful in itself for maintainability (it's easier to spot incongruities between the binning and the fill rule when they're right next to each other), it is also important for frameworks like Apache PySpark that distribute the fill operation. Without consolidating the fill logic like this, a PySpark `aggregate` call would be very difficult to maintain.

Finally, the last line

```python
roothist = histogram.root("name", "title")
```

creates a ROOT object from the Histogrammar object. In this case, it is a `ROOT.TH1D`, but the exact choice depends on what kind of Histogrammar object you're converting.

You can style and manipulate this ROOT object as you ordinarily would, using ROOT functions. The intention of Histogrammar is not to replace ROOT or any other plotting framework, but to provide an alternate means of aggregation that cuts across frameworks and encourages sharing of data, using each tool for what it does best.

## Composability

In the above, it might seem that `Bin` is Histogrammar's word for "histogram," but it is more general than that. The `Bin` constructor has several arguments with default values:

  * `value`: how to fill a bin between the low and high edges;
  * `underflow`: what to do with data below the low edge;
  * `overflow`: what to do with data above the high edge;
  * `nanflow`: what to do with data that is not a number (`NaN`).

ROOT would fill the value of each bin, as well as the underflow and overflow, with a count of entries. (Histogrammar additionally has a "nanflow" so that every input value fills _some_ bin.) This is the most common case, so Histogrammar has `Count()` as default values for these arguments.

Here, the similarity ends. Histogrammar's `Count` is an aggregator of the same sort as `Bin`, and they can be used interchangeably. You could have counted the data instead of binning it:

```python
count = Count()

for i, event in enumerate(events):
    if i == 1000: break
    count.fill(event)

print(count)
```

produces `<Count 1000.0>`, though this is only useful if you didn't already know how many elements you were looping over.

Let's try something more interesting: put a `Bin` of `Count` inside of a `Bin`.

```python
hist2d = Bin(10, -100, 100, lambda event: event.met.px,
             value = Bin(10, -100, 100, lambda event: event.met.py))

for i, event in enumerate(events):
    if i == 1000: break
    hist2d.fill(event)

roothist = hist2d.root("name2", "title")
roothist.Draw("colz")
```

![Two-dimensional histogram](hist2d.png)

A `Bin` of `Bin` of `Count` is a two dimensional histogram because the thing that we put inside each bin of the first histogram is another whole histogram. With just two primitives, we can express histograms of any dimension.

It's hard to visualize histograms with more than two dimensions, but we can still aggregate them. This kind of data reduction can be useful for things other than plotting. ROOT can view histograms up to three dimensions, so we can try it:

```python
hist3d = Bin(10, -100, 100, lambda muon: muon.px,
             Bin(10, -100, 100, lambda muon: muon.py,
                 Bin(10, -100, 100, lambda muon: muon.pz)))

for i, event in enumerate(events):
    if i == 1000: break
    for muon in event.muons:
        hist3d.fill(muon)

roothist = hist3d.root("name3", "title")
```

In Histogrammar version 0.7, at least, this raises

```
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'Bin' object has no attribute 'root'
```

meaning that Histogrammar does not have a known conversion from `Bin` of `Bin` of `Bin` of `Count` to a corresponding ROOT object. The types of histograms you can build with Histogrammar is literally infinite, but there is a finite set of patterns mapping Histogrammar objects onto objects in a given plotting library. The Histogrammar-ROOT front-end may someday handle three-dimensional histograms, but as of version 0.7 it does not.

Nevertheless, the aggregated data is available inside `hist3d`. Explore it with Python's `dir()` or tab-completion and come up with creative ways to visualize it.

```python
import ROOT
for i, x in enumerate(hist3d.values):
    low, high = hist3d.range(i)
    roothist = x.root("slice {}".format(i), "{} <= px < {}".format(low, high))
    roothist.Draw("colz")
    ROOT.gPad.SaveAs("slice_{}.png".format(i))

import os
os.system("convert -delay 100 -loop 0 slice*.png hist3d.gif")
```

![Three-dimensional histogram](hist3d.gif)

This could be improved with better binning, better ROOT styling (such as a fixed `colz` scale), and animated GIF conversion (pause before repeating), but you get the idea.

### Visual tour of Histogrammar primitives

We managed to produce three different visualizations with only `Count` and `Bin`, but there are two dozen different kinds of primitives to work with. Let's try a few more.

`Average` is a drop-in replacement for `Count` that averages a quantity instead of counting entries.

```python
average = Average(lambda event: event.met.pt)

for i, event in enumerate(events):
    if i == 1000: break
    average.fill(event)

print(average)
```

prints `<Average mean=26.5509706109>`.

If we bin it, we approximate a two-dimensional distribution as a function.


```python
pt_vs_vertices = Bin(20, 0.5, 20.5, lambda event: event.numPrimaryVertices,
                     Average(lambda event: event.met.pt))

events = EventIterator()
for i, event in enumerate(events):
    if i == 10000: break
    pt_vs_vertices.fill(event)

roothist = pt_vs_vertices.root("name4")
roothist.GetXaxis().SetTitle("number of primary vertices")
roothist.GetYaxis().SetTitle("average MET pT")
roothist.Draw()
```

![Profile without error bars](profile.png)

This is a profile plot (`roothist` is literally a `ROOT.TProfile`), which should be familiar to ROOT users. Profile plots usually have error bars, but the required information to make the error bar (variance in each bin) is not available because we only accumulated the averages.

A primitive called `Deviate` accumulates mean and variance:

```python
deviate = Deviate(lambda event: event.met.pt)

for i, event in enumerate(events):
    if i == 1000: break
    deviate.fill(event)

print(deviate)
```

prints `<Deviate mean=24.5114851815 variance=279.49272031>`. (Why the weird name? Every primitive in Histogrammar is a verb, and "variance" and "standard deviation" are nouns. Type-safe versions of Histogrammar&mdash; which does not include Python&mdash; use verb tenses to specify which stage of development the primitive is in. Python just uses duck typing.)

All we have to do to get error bars is to swap `Average` for `Deviate`.

```python
pt_vs_vertices = Bin(20, 0.5, 20.5, lambda event: event.numPrimaryVertices,
                     Deviate(lambda event: event.met.pt))

events = EventIterator()
for i, event in enumerate(events):
    if i == 10000: break
    pt_vs_vertices.fill(event)

roothist = pt_vs_vertices.root("name5")
roothist.GetXaxis().SetTitle("number of primary vertices")
roothist.GetYaxis().SetTitle("average MET pT")
roothist.Draw()
```

![Profile with error bars](profileerr.png)

It may be useful to know that every primitive has an `entries` (number of entries) field, and `Count` is nothing but a number of entries. Therefore, anything based on `Bin` can be turned into a simple histogram _without re-filling_.

```python
roothist2 = pt_vs_vertices.histogram().root("name6")
roothist2.Draw()
```

![Histogram from profile](hist_from_profileerr.png)

This can help you answer questions about your plots using data that you already have on-hand.

#### Alternate binning methods

`Count`, `Average`, and `Deviate` differ from `Bin` in one important aspect: they aggregate data, but do not pass it on to a sub-aggregator. If you compose aggregators into a tree, `Count`, `Average`, and `Deviate` will always be leaves and `Bin` will always be an internal node. This distinction is made in [the specification](../../specification) as primitives of "the first kind," "the second kind," etc. (The allusion to alien encounters is intentional.)

Peruse the specification to see what other "first kind" primitives you can make and think about how you'd be able to use things like quantiles and min/max per bin in an analysis. In this tutorial, we move on to alternate binning methods.

Standard binning slices a numeric interval into equal-length subintervals. The aggregator must do something with values that fall below (underflow) or above (overflow) the interval, and the data analyst has to guess an appropriate interval. It's not uncommon to make a histogram of, say, momentum, only to discover that the tail reaches higher than you expected (cutting it off) or lower than you expected (compressing the meaningful information into one or two bins). You then have to re-fill the histogram, which is aggregating when it happens frequently or takes a long time to fill.

If we replace the "dense vector" of a standard histogram with a "sparse vector," in which only non-zero values are filled, then there is no need to specify a minimum and maximum bin before filling. Consider the following:

```python
histogram = SparselyBin(1.0, lambda event: event.met.pt)

events = EventIterator()
for i, event in enumerate(events):
    if i == 1000: break
    histogram.fill(event)

roothist = histogram.root("name7", "title")
roothist.Draw()
```

![Sparse histogram](sparsehist.png)

This histogram was created with no prior knowledge that the minimum `event.met.pt` would be `0` and the maximum would be `93`. Only the bin width (the first argument, `1.0` momentum units per bin) had to be specified.

To see how this histogram differs, print the bins:

```python
print(histogram.bins)
```
```
{1: <Count 4.0>, 2: <Count 5.0>, 3: <Count 10.0>, 4: <Count 23.0>, 5: <Count 17.0>, 6: <Count 21.0>, 7: <Count 21.0>, 8: <Count 17.0>, 9: <Count 34.0>, 10: <Count 26.0>, 11: <Count 22.0>, 12: <Count 24.0>, 13: <Count 32.0>, 14: <Count 25.0>, 15: <Count 26.0>, 16: <Count 22.0>, 17: <Count 29.0>, 18: <Count 23.0>, 19: <Count 27.0>, 20: <Count 25.0>, 21: <Count 22.0>, 22: <Count 24.0>, 23: <Count 24.0>, 24: <Count 19.0>, 25: <Count 23.0>, 26: <Count 17.0>, 27: <Count 17.0>, 28: <Count 20.0>, 29: <Count 24.0>, 30: <Count 18.0>, 31: <Count 9.0>, 32: <Count 13.0>, 33: <Count 25.0>, 34: <Count 17.0>, 35: <Count 19.0>, 36: <Count 18.0>, 37: <Count 17.0>, 38: <Count 15.0>, 39: <Count 9.0>, 40: <Count 16.0>, 41: <Count 17.0>, 42: <Count 19.0>, 43: <Count 15.0>, 44: <Count 17.0>, 45: <Count 16.0>, 46: <Count 8.0>, 47: <Count 9.0>, 48: <Count 8.0>, 49: <Count 5.0>, 50: <Count 9.0>, 51: <Count 4.0>, 52: <Count 10.0>, 53: <Count 9.0>, 54: <Count 2.0>, 55: <Count 3.0>, 56: <Count 4.0>, 57: <Count 7.0>, 58: <Count 1.0>, 59: <Count 2.0>, 60: <Count 3.0>, 61: <Count 5.0>, 62: <Count 2.0>, 64: <Count 1.0>, 65: <Count 4.0>, 66: <Count 3.0>, 67: <Count 1.0>, 68: <Count 4.0>, 69: <Count 1.0>, 70: <Count 1.0>, 72: <Count 1.0>, 73: <Count 1.0>, 74: <Count 1.0>, 75: <Count 2.0>, 76: <Count 1.0>, 77: <Count 1.0>, 78: <Count 1.0>, 79: <Count 1.0>, 82: <Count 1.0>, 92: <Count 1.0>}
```

It's a Python dictionary, mapping non-empty bin indexes to `Count` objects. As new values are encountered, new bins are created, as well as a potentially new minimum and maximum.

```python
print(len(histogram.bins), histogram.low, histogram.high, histogram.minBin, histogram.maxBin)
```
```
(79, 1.0, 93.0, 1, 92)
```

```python
for i, event in enumerate(events):
    if i == 1000: break
    histogram.fill(event)

print(len(histogram.bins), histogram.low, histogram.high, histogram.minBin, histogram.maxBin)
```
```
(86, 0.0, 99.0, 0, 98)
```

Keep in mind that the memory use of a `Bin` is fixed, but a `SparselyBin` can grow without bound, depending on the distribution of the data. In particular, a distribution with a long tail might create a new bin for every tail event. Another concern is in converting a sparse histogram with far-flung values (such as `1e9` to represent "missing data") into a dense histogram: all the intermediate bins would be created with zero value.

These are usually not issues in exploratory data analysis, where sparse histograms are the most useful. Just be sure to check `histogram.minBin` and `histogram.maxBin` before attempting to plot it and replace `SparselyBin` with `Bin` and well-chosen `low` and `high` values in the final analysis.

That said, `SparselyBin` can be composed with any sub-aggregators to make sparse profile plots:

```python
pt_vs_vertices = SparselyBin(1.0, lambda event: event.numPrimaryVertices,
                             Deviate(lambda event: event.met.pt))

events = EventIterator()
for i, event in enumerate(events):
    if i == 10000: break
    pt_vs_vertices.fill(event)

roothist = pt_vs_vertices.root("name8")
roothist.GetXaxis().SetTitle("number of primary vertices")
roothist.GetYaxis().SetTitle("average MET pT")
roothist.Draw()
```

![Sparsely binned profile plot](sparseprof.png)

sparse two-dimensional histograms:

```python
hist2d = SparselyBin(5.0, lambda event: event.met.px,
                     SparselyBin(5.0, lambda event: event.met.py))

events = EventIterator()
for i, event in enumerate(events):
    if i == 1000: break
    hist2d.fill(event)

roothist = hist2d.root("name9", "title")
roothist.Draw("colz")
```

![Sparsely binned two-dimensional histogram](sparsehist2d.png)

and so on.

Notice that this last example has negative indexes (print `hist2d.bins.keys()`) and a `SparselyBin` nested within a `SparselyBin`.

Histogrammar has a few other binning methods:

   * `CentrallyBin`: a fixed set of irregularly spaced bins, defined by _bin centers._ Specifying the irregularly spaced bins by their centers has two nice features: no gaps and no underflow/overflow. It has an analogy with one-dimensional clustering. As of Histogrammar version 0.7, an automated translation to ROOT has not been defined.
   * `AdaptivelyBin`: the ultimate binning method for when you know nothing about the distribution: you specify a maximum number of bins and it adapts irregular bin centers to fit. It uses a one-dimensional clusering algrithm to adjust the bin centers, and the maximum memory use is finite (capped by the maximum number of bins). As of Histogrammar version 0.7, however, an automated translation to ROOT has not been defined.
   * `Partition`: could be used for irregularly spaced bins defined by _bin edges,_ though it was intended for groups of plots with different cuts (such as a series of different pseudorapidity cuts or heavy ion centrality bins).

Although `SparselyBin` requires more a priori knowledge (the bin width) than `AdaptivelyBin`, the fixed bin width and alignment of bins with round numbers are very useful in an analysis.

All of these are aggregators "of the second kind" in [the specification](../../specification).

#### Structures made of histograms

If "first kind" primitives like `Count`, `Average`, and `Deviate` are like stars and binning methods like `Bin`, `SparselyBin`, and `AdaptivelyBin` are like clusters of stars, there are yet larger structures like galaxies and galactic clusters. Some of the things we do in an analysis involve a coordinated use of multiple histograms.

One of the most common is an efficiency plot: the probability of passing a cut as a function of some binned variable. You could make this by filling two histograms, one with the cut, the other without, and then dividing them, but Histogrammar has a built-in primitive:

```python
frac = Fraction(lambda event: event.numPrimaryVertices > 5,
                Bin(30, 0, 100, lambda event: event.met.pt))

events = EventIterator()
for i, event in enumerate(events):
    if i == 10000: break
    frac.fill(event)

roothist = frac.root("name10", "title")
roothist.SetTitle(";MET pT;fraction with number of primary vertices > 5")
roothist.Draw()
```

![Efficiency plot](frac.png)

Note that the ROOT object returned by `frac.root` is a `ROOT.TEfficiency`. This lets you choose different statistics for the error bars, which is not Histogrammar's job, but ROOT's.

There's nothing stopping you from making an efficiency plot from a sparsely binned histogram. Histogrammar ensures that the numerator and denominator have the same binning.

```python
frac = Fraction(lambda event: event.numPrimaryVertices > 5,
                SparselyBin(5.0, lambda event: event.met.pt))

for i, event in enumerate(events):
    if i == 10000: break
    frac.fill(event)

roothist = frac.root("name11", "title")
roothist.SetTitle(";MET pT;fraction with number of primary vertices > 5")
roothist.Draw()
```

![Sparsely binned efficiency plot](sparsefrac.png)

In fact, you could compute the fraction of anything, even a simple count.




<!--


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

events = EventIterator()
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

events = EventIterator()
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
  * [`Average`](../..//specification/#average-mean-of-a-quantity) and [`Deviate`](../..//specification/#deviate-mean-and-variance): add mean and variance, cumulatively.
  * [`Minimize`](../..//specification/#minimize-minimum-value) and [`Maximize`](../..//specification/#maximize-maximum-value): lowest and highest value seen.

**Histogram-like objects:**

  * [`Bin`](../..//specification/#bin-regular-binning-for-histograms) and [`SparselyBin`](../..//specification/#sparselybin-ignore-zeros): split a numerical domain into uniform bins and redirect aggregation into those bins.
  * [`Categorize`](../..//specification/#categorize-string-valued-bins-bar-charts): split a string-valued domain by unique values; good for making bar charts (which are histograms with a string-valued axis).
  * [`Partition`](../..//specification/#partition-exclusive-filling): split a numerical domain into arbitrary subintervals, usually for separate plots like particle pseudorapidity or collision centrality.

**Collections:**

  * [`Label`](../../specification/#label-directory-with-string-based-keys), [`UntypedLabel`](../..//specification/#untypedlabel-directory-of-different-types), and [`Index`](../..//specification/#index-list-with-integer-keys): bundle objects with string-based keys (`Label` and `UntypedLabel`) or simply an ordered array (effectively, integer-based keys) consisting of a single type (`Label` and `Index`) or any types (`UntypedLabel`).
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

  * [`Bag`](../../specification/#bag-accumulate-values-for-scatter-plots) and [`Sample`](../..//specification/#sample-reservoir-sampling): collect data points, rather than aggregate quantities.





from histogrammar import *
histogram2 = Select(unweighted, Bin(91, 1.0, 93.0, lambda event: event.met.pt))
histogram2 = Histogram(91, 1.0, 93.0, lambda event: event.met.pt)
events = EventIterator()
for i, event in enumerate(events):
    if i == 1000: break
    histogram2.fill(event)

roothist2 = histogram2.root("name10", "title")
roothist2.Draw()

from histogrammar import *
histogram = Select(unweighted, SparselyBin(1, lambda event: event.met.pt))
histogram = SparselyHistogram(1, lambda event: event.met.pt)
events = EventIterator()
for i, event in enumerate(events):
    if i == 1000: break
    histogram.fill(event)

roothist = histogram.root("name11", "title")
roothist.Draw()

from histogrammar import *
import math
pt_vs_phi = Select(unweighted, Bin(30, -math.pi, math.pi, lambda event: math.atan2(event.met.py, event.met.px), Average(lambda event: event.met.pt)))
pt_vs_phi = Profile(30, -math.pi, math.pi, lambda event: math.atan2(event.met.py, event.met.px), lambda event: event.met.pt)
events = EventIterator()
for i, event in enumerate(events):
    if i == 100000: break
    pt_vs_phi.fill(event)

roothist = pt_vs_phi.root("name12", "title")
roothist.Draw()

from histogrammar import *
import math
pt_vs_phi = Select(unweighted, Bin(30, -math.pi, math.pi, lambda event: math.atan2(event.met.py, event.met.px), Deviate(lambda event: event.met.pt)))
pt_vs_phi = Profile(30, -math.pi, math.pi, lambda event: math.atan2(event.met.py, event.met.px), lambda event: event.met.pt)
events = EventIterator()
for i, event in enumerate(events):
    if i == 100000: break
    pt_vs_phi.fill(event)

roothist = pt_vs_phi.root("name13", "title")
roothist.Draw()

from histogrammar import *
import math
pt_vs_phi = Select(unweighted, SparselyBin(2.0*math.pi/30.0, lambda event: math.atan2(event.met.py, event.met.px), Average(lambda event: event.met.pt)))
pt_vs_phi = SparselyProfile(2.0*math.pi/30.0, lambda event: math.atan2(event.met.py, event.met.px), lambda event: event.met.pt)
events = EventIterator()
for i, event in enumerate(events):
    if i == 1000: break
    pt_vs_phi.fill(event)

roothist = pt_vs_phi.root("name14", "title")
roothist.Draw()

from histogrammar import *
import math
pt_vs_phi = Select(unweighted, SparselyBin(2.0*math.pi/30.0, lambda event: math.atan2(event.met.py, event.met.px), Deviate(lambda event: event.met.pt)))
pt_vs_phi = SparselyProfileErr(2.0*math.pi/30.0, lambda event: math.atan2(event.met.py, event.met.px), lambda event: event.met.pt)
events = EventIterator()
for i, event in enumerate(events):
    if i == 1000: break
    pt_vs_phi.fill(event)

roothist = pt_vs_phi.root("name", "title")
roothist.Draw()

from histogrammar import *
import math
pt_vs_phi = Select(unweighted, Bin(30, -math.pi, math.pi, lambda event: math.atan2(event.met.py, event.met.px), Bin(20, 0, 100, lambda event: event.met.pt)))
pt_vs_phi = TwoDimensionallyHistogram(30, -math.pi, math.pi, lambda event: math.atan2(event.met.py, event.met.px), 20, 0, 100, lambda event: event.met.pt)
events = EventIterator()
for i, event in enumerate(events):
    if i == 100000: break
    pt_vs_phi.fill(event)

roothist = pt_vs_phi.root("name", "title")
roothist.Draw("colz")

from histogrammar import *
import math
pt_vs_phi = Select(unweighted, SparselyBin(2.0*math.pi/30.0, lambda event: math.atan2(event.met.py, event.met.px), SparselyBin(5.0, lambda event: event.met.pt)))
pt_vs_phi = TwoDimensionallySparselyHistogram(2.0*math.pi/30.0, lambda event: math.atan2(event.met.py, event.met.px), 5.0, lambda event: event.met.pt)
events = EventIterator()
for i, event in enumerate(events):
    if i == 1000: break
    pt_vs_phi.fill(event)

roothist = pt_vs_phi.root("name", "title")
roothist.Draw("colz")

from histogrammar import *
import math
hist = Select(unweighted, SparselyBin(5, lambda event: event.met.px, SparselyBin(5, lambda event: event.met.py)))
hist = TwoDimensionallySparselyHistogram(5, lambda event: event.met.px, 5, lambda event: event.met.py)
events = EventIterator()
for i, event in enumerate(events):
    if i == 1000: break
    hist.fill(event)

roothist = hist.root("name", "title")
roothist.Draw("colz")



from histogrammar import *
fraction = Fraction(lambda event: event.numPrimaryVertices > 5, Select(unweighted, Bin(91, 1.0, 93.0, lambda event: event.met.pt)))
fraction = Fraction(lambda event: event.numPrimaryVertices > 5, Histogram(91, 1.0, 93.0, lambda event: event.met.pt))
events = EventIterator()
for i, event in enumerate(events):
    if i == 10000: break
    fraction.fill(event)

roothist = fraction.root("x", "y")
roothist.Draw()


from histogrammar import *
fraction = Fraction(lambda event: event.numPrimaryVertices > 5, SparselyBin(1, lambda event: event.met.pt))
events = EventIterator()
for i, event in enumerate(events):
    if i == 10000: break
    fraction.fill(event)

roothist = fraction.root("x", "y")
roothist.Draw()





from histogrammar import *
fraction = Stack([5, 10, 15, 20], lambda event: event.numPrimaryVertices, Select(unweighted, Bin(91, 1.0, 93.0, lambda event: event.met.pt)))
fraction = Stack([5, 10, 15, 20], lambda event: event.numPrimaryVertices, Histogram(91, 1.0, 93.0, lambda event: event.met.pt))
events = EventIterator()
for i, event in enumerate(events):
    if i == 10000: break
    fraction.fill(event)

roothist = fraction.root("one", "two", "three", "four", "five")
roothist.Draw()

from histogrammar import *
fraction = Stack([5, 10, 15, 20], lambda event: event.numPrimaryVertices, Select(unweighted, SparselyBin(1, lambda event: event.met.pt)))
fraction = Stack([5, 10, 15, 20], lambda event: event.numPrimaryVertices, SparselyHistogram(1, lambda event: event.met.pt))
events = EventIterator()
for i, event in enumerate(events):
    if i == 10000: break
    fraction.fill(event)

roothist = fraction.root("one", "two", "three", "four", "five")
roothist.Draw()



from histogrammar import *
fraction = Partition([5, 10, 15, 20], lambda event: event.numPrimaryVertices, Select(unweighted, Bin(91, 1.0, 93.0, lambda event: event.met.pt)))
fraction = Partition([5, 10, 15, 20], lambda event: event.numPrimaryVertices, Histogram(91, 1.0, 93.0, lambda event: event.met.pt))
events = EventIterator()
for i, event in enumerate(events):
    if i == 10000: break
    fraction.fill(event)

roothist = fraction.root("one", "two", "three", "four", "five")
roothist.Draw()


from histogrammar import *
fraction = Partition([5, 10, 15, 20], lambda event: event.numPrimaryVertices, Select(unweighted, SparselyBin(1, lambda event: event.met.pt)))
fraction = Partition([5, 10, 15, 20], lambda event: event.numPrimaryVertices, SparselyHistogram(1, lambda event: event.met.pt))
events = EventIterator()
for i, event in enumerate(events):
    if i == 10000: break
    fraction.fill(event)

roothist = fraction.root("one", "two", "three", "four", "five")
roothist.Draw()






-->
