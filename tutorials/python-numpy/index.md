---
title: Accelerating histogram-filling with Numpy
type: default
toc: false
summary: |
    <p>If you're working with in Python and want to fill your histograms with <a href="http://www.numpy.org/">Numpy</a> arrays, read this page.</p>
    <p><b>Author:</b> <a href="http://github.com/jpivarski">Jim Pivarski</a></p>
---

## Preliminaries

This tutorial assumes familiarity with using Histogrammar in Python. It only shows how to accelerate your histogram filling with Numpy.

It should work with Histogrammar version 0.8 and above.

## Quick example

If you want to fill your histograms with a data from a Numpy array named `arr`, you might be tempted to do this:

```python
from histogrammar import *

histogram = Bin(100, 0, 10, lambda x: x**2)  # or any other aggregator

for x in arr:
    histogram.fill(arr)
```

It would be much faster if you instead did this:

```python
histogram.numpy(arr)
```

The result would be the same (`histogram` is now filled), but depending on the complexity of `histogram`, the outcome could be achieved ten to a hundred times sooner.

Due to duck typing and Numpy's overloaded operators, the `lambda x: x**2` that transforms scalar values in the first case transforms whole Numpy arrays in the second case.

## Numpy overview and motivation

Numpy is fast because it does less work: its arrays have a fixed size and a fixed type, such as all 32-bit integers or all 64-bit floats. Unlike Python's for loop, Numpy's internal loops do not have to check the type of each item before each operation. Also, its arrays are contiguous in memory with some attempt to stride them for cache efficiency, which is the overriding performance consideration for modern CPUs. (Memory access is the bottleneck in modern CPUs, not computation, so good cache utilization is essential.) Python references, even for primitive types like integers and floating-point numbers, are pointers that could reside anywhere in memory.

(Of course it's possible to thwart this efficiency: a Numpy array with `dtype=object` operates on Python objects and does not provide as much of a speed-up.)

The downside is that converting a pure Python analysis to Numpy requires you to fundamentally rethink your algorithm in many cases. In pure Python, for loops usually contain more than one operation, but each Numpy operation has a single for loop embedded within it. Let's start with an easy example to illustrate this. You have two Python lists, `xlist` and `ylist`, and you want to find the Euclidean distance of each `x`, `y` pair from the origin.

```python
result = []
for x, y in zip(xlist, ylist):
    result.append(math.sqrt(x**2 + y**2))
```

This is, I believe, the most "natural" way to think about it. You're thinking about events independently (an `x`, `y` pair is like an "event"), and you write down what you want to do to each of them. It's slow because Python doesn't know how large to allocate `result`, so it keeps reallocating as results accumulate. Then the for loop constructs and then deconstructs a two-tuple to define each pair, and Python has to verify that the values are numbers when it squares them, again when it adds them, and again when it computes the square root. _We_ know that all this extra work is not necessary, but Python doesn't.

The Numpy version looks pretty similar:

```python
result = numpy.sqrt(xarray**2 + yarray**2)
```

but what it's doing is fundamentally different. Numpy squares all elements of `xarray`, making a new, unnamed, temporary array `xarray**2`. Then it does the same with `yarray` to make `yarray**2`. Then it adds the two temporary arrays, and finally it computes the square root of each element (note that this is `numpy.sqrt` and not `math.sqrt`). Numpy has allocated a lot of temporary arrays, but in the end, it's 37 times faster than the pure Python version (on my laptop with a million-element list).

If we think of our data schematically as a table with two columns (`x` and `y`) and a million rows, the pure Python implementation steps row-by-row and computes the full expression individually. The Numpy implementation computes each operation of the expression a whole column (or temporary column) at a time. The Numpy syntax is great because it makes the fast column-wise expression look just like the slow row-wise expression.

However, this syntax doesn't work if the calculation is anything but a table transformation (same number of output rows, each row is transformed independently). Numpy has a nice syntax for filtering rows (with square brackets, like R), and this allows for the full complexity of SQL with `SELECT` and `WHERE` but no `GROUP BY`. Here are some things that are hard to translate unless there is a Numpy function that does exactly what you want:

   * Rows containing arbitrary length objects, such as events containing lists of jets, or events that have <i>N</i> tracks that have <i>M<sub>n</sub></i> hits.
   * Iterative procedures on each row. If a procedure is supposed to iterate until a condition defined on a row is true, the Numpy version will have to iterate until it's true on _all_ rows. (You could do this by performing unnecessary iterations on the completed rows or by allocating smaller subarrays of survivors with each iteration. Which one is faster depends on the straggler distribution.)
   * Operations that mix data from different rows.
   * Data reductions, such as aggregation (SQL's `GROUP BY`). Simple cases like `numpy.sum` are part of the Numpy library, but a general reducer taking a Python lambda function, like Spark's `aggregate`, would lose most of the Numpy performance because it would have to call out to Python on each row.

Reductions like choosing histogram bins using one Numpy array and computing averages in each bin with another become very complicated:

```python
# Get a bin index for each x in xarray.
xindex = numpy.array(numpy.floor(
    num * (xarray - low) / (high - low)), dtype=int)

profiles = []

# For each bin,
for index in range(num):
    # find the y values that fall in this bin,
    yinbin = yarray[xindex == index]

    # and average them.
    profiles.append(yinbin.mean())

# Now look at the profile plot.
print profiles
```

Numpy has a built-in `numpy.histogram` for the simple case (count in each bin), but not for anything more complicated (e.g. average in each bin).

## Histogrammar in Numpy

This is where Histogrammar comes in. Histogrammar describes complex aggregations in a declarative way, so although its generic implementation operates row-by-row, it can just as easily be computed column-by-column. A pure Python workflow built out of nested Histogrammar primitives can be Numpy-enhanced with little or no alteration to the code.

For instance, if we want to fill a histogram with Euclidean distances, the pure Python approach would be

```python
# data = [{"x": x0, "y": y0}, {"x": x1, "y": y1}, ...]

histogram = Bin(100, 0, 5, lambda datum: math.sqrt(datum["x"]**2 + datum["y"]**2))

for datum in data:
    histogram.fill(datum)

print histogram.numericalValues
```

and the Numpy approach would be

```python
# data = {"x": numpy.array([x0, x1, ...]), "y": numpy.array([y0, y1, ...])}

histogram = Bin(100, 0, 5, lambda data: numpy.sqrt(data["x"]**2 + data["y"]**2))

histogram.numpy(data)

print histogram.numericalValues
```

It's the same construction of Histogrammar primitives (just a `Bin` here, but it could have been any complex structure.) What differs is the structure of the input data passed as arguments to the lambda functions and the return values of the lambda functions.

In the pure Python case, we have a big list of individual rows and the lambda takes one row (`datum`) each time it is invoked (many times). In the Numpy case, we have a single dict of column arrays and the lambda takes the whole collection when it is invoked (once).

Histogrammar's string-based shorthand for function definitions hides this distinction:

```python
histogram = Bin(100, 0, 5, "sqrt(x**2 + y**2)")
for datum in data:
    histogram.fill(datum)
```

```python
histogram = Bin(100, 0, 5, "sqrt(x**2 + y**2)")
histogram.numpy(data)
```

The string gets compiled into a Python function using `math.sqrt` for `sqrt` if you don't have Numpy and `numpy.sqrt` if you do. Given `x` and `y` as floating point scalars, Python will compute a floating point scalar, and given `x` and `y` as Numpy arrays, Python will compute a Numpy array. The fact that we passed in a `datum` in the first case and `data` in the second determines how the function will be evaluated, and the fact that we called `fill` in the first case and `numpy` in the second determines how Histogrammar will attempt to use that result.

If the wrong type is used in either case, Histogrammar will raise an error: `fill` requires lambda functions to produce scalars, and `numpy` requires lambda functions to produce one-dimensional Numpy arrays. (Histogrammar will also raise errors if Numpy arrays at any level of the aggregation have different lengths. They're supposed to represent a single, rectangular table.)

## Generality of the interface

In my example above, the Numpy implementation extracts all data from a dict of Numpy columns. It could have been anything else. The job of producing one-dimensional Numpy arrays from `data`, whatever `data` happens to be, is up to the lambda functions. If you have a weird data source, you need correspondingly weird lambda functions.

However, the idea of having some string-key index to look up equal-length, one-dimensional columns of numbers is a pretty general one. It's how I do most of my Numpy-based analysis, and it's also how Numpy's [built-in structured arrays](http://docs.scipy.org/doc/numpy/user/basics.rec.html) are presented to the user. In addition, it's the same rough idea as [Pandas DataFrames](http://pandas.pydata.org/), so all of the following work with the same function string passed to Histogrammar:

```python
import random
import numpy

class Data:
    def __init__(self, x, y):
        self.x = x
        self.y = y

xarray = numpy.array([random.gauss(0, 1) for i in xrange(1000000)])
yarray = numpy.array([random.gauss(0, 1) for i in xrange(1000000)])

# dict of arrays
histogram = Bin(100, 0, 5, "sqrt(x**2 + y**2)")
histogram.numpy({"x": xarray, "y": yarray})

# object of arrays
histogram = Bin(100, 0, 5, "sqrt(x**2 + y**2)")
histogram.numpy(Data(xarray, yarray))

# Numpy record array
r = numpy.rec.array((xarray, yarray), names=["x", "y"])
histogram = Bin(100, 0, 5, "sqrt(x**2 + y**2)")
histogram.numpy(r)

# Pandas DataFrame
df = pandas.DataFrame({"x": xarray, "y": yarray})
histogram = Bin(100, 0, 5, "sqrt(x**2 + y**2)")
histogram.numpy(df)
```

More could be added (SQLAlchemy?) to Histogrammar's inspection rules. These libraries are not dependencies; if they're not available on your system, Histogrammar will silently not attempt to use them.

## Performance and coverage

As mentioned in the beginning of this tutorial, the speedup depends on the complexity of the aggregator. A histogram with many bins could even be slower in Numpy because the whole dataset needs to be examined to find the subset matching each individual bin. (But not always: normal histograms, i.e. `Bin` of `Count`, use `numpy.histogram` as a shortcut and therefore scale better than `Bin` of arbitrary content. This shortcut also requires all-finite values: no `inf`, `-inf`, or `nan`.)

Also, if the dataset is small or empty, Numpy may lose due to time spent setting up and tearing down the calculation. As usual, optimization is driven by particular cases, which is why it's good that the same Histogrammar construct runs in each mode. You can test both without rewriting your analysis, which is not the case for general Numpy translations!

Here is a rough table of speedup factors (wall time for pure Python divided by wall time for Numpy) for primitives in typical applications. For instance, it's possible to use `Stack` on a suite of `Counts`, but it's usually used on a suite of  (`Bin` of `Count`).

| Primitive         | Numpy speedup | Primitive         | Numpy speedup |
|:------------------|:-----------------|:------------------|:-----------------|
| [Count](../../specification/#count-sum-of-weights)                                  | ~100X                 | [Categorize](../../specification/#categorize-string-valued-bins-bar-charts)         | 1.5X             |
| [Sum](../../specification/#sum-sum-of-a-given-quantity)                             | 40-100X               | [Fraction](../../specification/#fraction-efficiency-plots)                          | 4-20X (100 bins) |
| [Average](../../specification/#average-mean-of-a-quantity)                          | 40-100X               | [Stack](../../specification/#stack-cumulative-filling)                              | 2-12X (10 plots) |
| [Deviate](../../specification/#deviate-mean-and-variance)                           | 40-80X                | [Select](../../specification/#select-apply-a-cut)                                   | 4-20X (100 bins) |
| [Minimize](../../specification/#minimize-minimum-value)                             | 50-150X               | [Limit](../../specification/#limit-keep-detail-until-entries-is-large)              | pass-through     |
| [Maximize](../../specification/#maximize-maximum-value)                             | 50-150X               | [Label](../../specification/#label-directory-with-string-based-keys)                | pass-through     |
| [Bin](../../specification/#bin-regular-binning-for-histograms)                      | 5-25X (for 100 bins)  | [UntypedLabel](../../specification/#untypedlabel-directory-of-different-types)      | pass-through     |
| [SparselyBin](../../specification/#sparselybin-ignore-zeros)                        | 4-5X (about 100 bins) | [Index](../../specification/#index-list-with-integer-keys)                          | pass-through     |
| [CentrallyBin](../../specification/#centrallybin-fully-partitioning-with-centers)   | 25-40X (10 bins)      | [Branch](../../specification/#branch-tuple-of-different-types)                      | pass-through     |
| [IrregularlyBin](../../specification/#irregularlybin-fully-partitioning-with-edges) | 1-4X (10 plots)       | [Bag](../../specification/#bag-accumulate-values-for-scatter-plots)                 | 1.5-2X           |

**Some caveats:**

   * `Count` cannot be used alone in Numpy because it does not extract a quantity, a one-dimensional array to examine and get the length. `Counts` embedded in primitives that extract quantities are fine because those other primitives determine (and assert!) the lengths of the arrays.
   * `Bin` has a wide range of speedups because some cases can use `numpy.histogram` and others can't. (See above.)
   * `Categorize` operates on strings and builds a dictionary: not something you can do in Numpy, so it falls back on the pure Python implementation.
   * `Label`, `UntypedLabel`, `Index`, and `Branch` are just containers; they pass the data along in the same way whether the data fill row-by-row or column-by-column.
   * `Bag` defers to its pure Python implementation.
