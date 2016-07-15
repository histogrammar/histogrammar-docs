---
title: Specification 0.8
type: default
toc: true
summary: |
    <p>This page describes Histogrammar in detail, but without reference to any particular implementation, including the composable primitives, their required functions and argument lists, and their JSON representations.</p>
    
    <p>This is the normative specification; any implementations that don't adhere to the definitions on this page should be corrected.</p>

    <p>Previous versions: <a href="0.7">0.7</a></p>
---

# General features

Histogrammar consists of twenty aggregation primitives, each performing a simple task, that can be composed to solve complex aggregation problems. These primitives share the following properties.

### Reducers

The aggregators, individually and collectively, are reducers in the map-reduce sense. That is, they reduce a large table of data points to a small data structure that summarizes the whole. For instance, a traditional histogram is an array of counts, produced by incrementing array bins for each data point associated with it, and the resulting data structure approximates the probability density of the dataset in a lossy-compressed form.

### Composable

A primitive that accepts a sub-aggregator as an argument to its constructor may accept _any_ sub-aggregator. The suite of primitives should be thought of as building blocks to compute whatever complex statistics are needed.

### Commutative Monoids

Every primitive has a `combine` method, usually presented as the `+` operator, that allows aggregations of data partitions to be combined into an aggregation of the whole dataset. This operation is associative, commutative, and has an identity (the aggregator's initial state). Such an algebra is known as a _commutative monoid_, and it allows aggregation jobs to be distributed arbitrarily: the dataset may be partitioned in any way that is convenient, the data accumulated in any order, and the final result is always the same.

### Serializable

Every filled aggregator has a language-independent JSON representation, allowing results to be transmitted across networks, co-processors, frameworks, and languages. JSON was chosen because aggregated data is supposed to be a _small_ representation of the whole. For large aggregates, consider binary JSON alternatives (such as Apache Avro).

### Verbs

Each type of primitive has two forms:

  * the "present tense," which knows how to fill itself;
  * the "past tense," which has lost this information due to being serialized and reconstituted.

When constructing an aggregator, the data analyst defines functions for extracting the relevant quantities from the data. A present-tense aggregator uses these functions in its `fill` method to update its state. After serializing to and from JSON, an aggregator may have crossed to an entirely new language where the fill functions are no longer meaningful. It retains its data as a past-tense aggregator that can be combined but not filled.

As a mnemonic, each primitive is named as a verb, with the present-tense "-ing" form for mutable, fillable aggregators and the past-tense "-ed" form for immutable, filled aggregators. The infinitive is used for the factory that constructs both forms (if the factory pattern is used, rather than constructors). Dynamically typed languages like Python and R may use the same class for present and past tenses (named with the infinitive) with different sets of fields for the two cases.

### Exception-safe

The `fill` methods only update an aggregator's state _after_ the user-defined functions have all been evaluated. An exception in a user-defined function leaves a complex aggregator in its state prior to the fill attempt, not an inconsistent state.

## Version numbers

Specifications are released with two digit version numbers: _major.minor._ Implementations are released with three digit version numbers: _major.minor.bugfix._

The major and minor version numbers of implementations must match the major and minor numbers of the specification they implement. Bugfix numbers increase as errors are corrected and functions outside the scope of the specification are added.

For example, Histogrammar-Python 0.8.5 should be adhere to version 0.8 of the specification and therefore be interoperable with Histogrammar-Scala 0.8.3. Always use the latest bugfix version, but pick a _major.minor_ version for compatibility.

Starting in version 1.0, all versions of the specification with the same major number are backward compatible. That is, code using Histogrammar 1.5 interfaces will still work in Histogrammars 1.6, 1.7, and 1.8, but maybe not 2.0 and maybe not 1.4.

## Data model

The input data for all aggregators is taken to be a stream, possibly an infinite stream, of _entries._ User-defined functions map an entry to something an aggregator can use, such as a boolean for filtering, a floating point number for computing a mean or selecting a bin, or a string for categorical data.

A composite aggregator may contain many plotable data structures, such as histograms, profile plots, and scatter plots, but they must all be filled by a single data source. For instance, data from one sample can be plotted hundreds of different ways within a composite aggregator, revealing subtle relationships in that one dataset, but it cannot be compared with data in a qualitatively different sample (a table with different columns). For that, the data analyst needs to construct independent aggregators and run separate jobs.

It may be possible to merge data from multiple sources with the equivalent of an SQL `JOIN` operation before it enters Histogrammar, but in that case, Histogrammar's source is still a single stream.

## Data weights

Selection functions in Histogrammar are "fuzzy." While they may entirely keep or entirely drop entries, they can also keep an entry with partial weight, such as 0.5 (half), 0.1 (tenth), or 1.5 (overweight). A weight of 0.0 is equivalent to dropping an entry, a weight of 1.0 is equivalent to not weighting, and a weight of 2.0 is equivalent to filling with two identical events, each with weight 1.0. Negative and NaN weights are treated as 0.0 and ignored. The total weight can be positive infinity, but it can never be NaN or negative. All primitives interpret weights this same way.

When two selection functions are nested, their weights multiply. Different parts of a composite aggregator may apply different selections, but all of the selections applied to a particular primitive can be deduced by following the path from the root of the tree to that primitive. Composite aggregators are in this way self-documenting.

Hisogrammar weights are metadata: though they are calculated from the data in user-defined functions, they are passed from one `fill` method to the next separately from the data entries themselves. Therefore, if a statistical procedure needs to make use of "negative weights," it can do so by calculating its own weights, which is simply data from Histogrammar's point of view. For example, a Count primitive accumulates the sum of Histogrammar weights, which are always non-negative. To accumulate the sum of possibly negative weights, replace Count with Sum, and give the Sum an appropriate lambda function.

## User-defined functions

Each language has its own way of declaring functions; Histogrammar only assumes that functions (of some form) can be passed as arguments. For instance, Java technically does not have first class functions, but it can define objects in place with an `apply` method.

Histogrammar implementations must also be able to assign names to the functions as strings known at runtime. These names may be treated as metadata, carried separately from the functions themselves. When an aggregator is serialized as JSON, the functions are lost but their names are passed through for bookkeeping. For instance, a data analyst may define a function that extracts a distance in centimeters, label it as "x (cm)", and use it in a dozen histograms. After aggregating, serializing, transmitting, and reconstituting the histograms, they are still labeled as "x (cm)" and this label may be used on the axis of a plot.

Moreover, if the analyst ever changes the units, introducing a factor of 10 in the function and changing the label to "x (mm)", this calculation and its label are correctly updated in all the histograms in which it was used. The same would not be true if all histograms that use "x (mm)" were labeled independently.

## JSON representation

Every primitive has a JSON _fragment_ representation, which are nested in JSON the same way the primitives themselves are nested. The fragment does not carry information about the primitive type; this information is encoded as a field in the fragment's parent. This reduces redundancy in the serialized form, since containers might store large collections of sub-aggregators of the same type. The top of the tree wraps the topmost fragment in a JSON object containing

  * `type` (JSON string), name of the topmost primitive type (infinitive)
  * `data`, the fragment itself.

```json
{"type": TYPENAME, "data": FRAGMENT}
```

Thus, serializing or deserializing a whole Histogrammar document is distinct from serializing or deserializing any fragment. The whole-document serialization/deserialization functions must be available to the user, but the fragment functions may be hidden as an implementation detail.

JSON fragments for primitives that accept a function have an optional `"name"` attribute in their JSON representation to carry the function name. To reduce redundancy, collections of aggregators with the same name are named once in the parent as `"values:name"` or similar. Serialization of an aggregator should not produce `"values:name"` in the parent and `"name"` in the children, but if this is encountered in deserialization, the child's `"name"` overrides the parent's `"values:name"`. Examples are included in the documentation below.

## Conventions used in this specification

In the rest of this document, constructor arguments, required members, `fill`, `combine` algorithms, and JSON representations are given for each type of aggregator primitive. Histogrammar implementations should reproduce these names, calling structure, and data types exactly, though of course the syntax differs from one language to another.

Implementations match if they produce the same numerical results with a relative and absolute tolerance of 10<sup>-12</sup>, exactly agree on NaN and infinite cases, exactly agree on strings/null/boolean values, produce lists in the same order, and maps with the same key-value pairs in any order. Although IEEE double precision is the standard for floating-point numbers in Histogrammar, some environments like GPUs may have only single precision floats or premultiplied integers, requiring a looser (and documented) tolerance. Results must match regardless of how the input entries are partitioned (associativity) or shuffled (commutativity).

The `fill` and `combine` algorithms are expressed in Python syntax for concreteness, with a goal of clarity, not performance. (In many cases, the Python version of Histogrammar is implemented differently for performance). The fact that each primitive has required members does not preclude implementations from defining more member functions and member data, even with semi-standard names and argument lists across implementations. But they are not _required._

Also, some arguments are represented here as lists of values. If the language allows it, they may be varargs. Histogrammar implementations have a preference for varargs over list arguments.

If this document disagrees with the behavior of a Histogrammar implementation, this document should be taken as normative and the implementation should be corrected.

Happy histogramming!

# Zeroth kind: depend only on weights
    
## **Count:** sum of weights

Count entries by accumulating the sum of all observed weights or a sum of transformed weights (e.g. sum of squares of weights).

An optional `transform` function can be applied to the weights before summing. To accumulate the sum of squares of weights, use `lambda x: x**2`, for instance. This is unlike any other primitive's `quantity` function in that its domain is the _weights_ (always floating-point numbers), not _entries_ (any type).

### Counting constructor and required members

```python
Count.ing(transform=identity)
```

  * `transform` (function from double to double) transforms each weight.
  * `entries` (mutable double) is the number of entries, initially 0.0.

### Counted constructor and required members

```python
Count.ed(entries)
```

  * `entries` (double) is the number of entries.

### Fill and combine algorithms

```python
identity = lambda x: x

def fill(counting, datum, weight):
    if weight > 0.0:
        counting.entries += counting.transform(weight)

def combine(one, two):
    return Count.ed(one.entries + two.entries)
```

### JSON fragment format

Simply a JSON number (or JSON string "inf" for infinite values) representing the `entries`. This is the only aggregator whose representation is not a JSON object.

**Example:**

Here is a stand-alone Histogrammar document representing one Count.

```json
{"type": "Count", "data": 123.0}
```

And here is Count in a more typical context: embedded as values (and underflow, overflow, nanflow) of a [Bin](#bin-regular-binning-for-histograms).

```json
{"type": "Bin",
 "data": {
   "low": -5.0,
   "high": 5.0,
   "entries": 613.0,
   "values:type": "Count",
   "values": [10.0, 20.0, 20.0, 30.0, 30.0, 40.0, 50.0, 60.0, 70.0, 80.0, 90.0, 100.0],
   "underflow:type": "Count", "underflow": 5.0,
   "overflow:type": "Count", "overflow": 8.0,
   "nanflow:type": "Count", "nanflow": 0.0}}
```

# First kind: aggregate data without sub-aggregators

## **Sum:** sum of a given quantity

Accumulate the (weighted) sum of a given quantity, calculated from the data.

Sum differs from [Count](#count-sum-of-weights) in that it computes a quantity on the spot, rather than percolating a product of weight metadata from nested primitives. Also unlike weights, the sum can add both positive and negative quantities (weights are always non-negative).

### Summing constructor and required members

```python
Sum.ing(quantity)
```

  * `quantity` (function returning double) computes the quantity of interest from the data.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `sum` (mutable double) is the running sum, initially 0.0.

### Summed constructor and required members

```
Sum.ed(entries, sum)
```

  * `entries` (double) is the number of entries.
  * `sum` (double) is the sum.

### Fill and combine algorithms

```python
def fill(summing, datum, weight):
    if weight > 0.0:
        q = summing.quantity(datum)
        summing.entries += weight
        summing.sum += q * weight

def combine(one, two):
    return Sum.ed(one.entries + two.entries, one.sum + two.sum)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `sum` (JSON number, "nan", "inf", or "-inf")
  * optional `name` (JSON string), name of the `quantity` function, if provided.

**Example:**

```json
{"type": "Sum",
 "data": {"entries": 123.0, "sum": 3.14, "name": "myfunc"}}
```

## **Average:** mean of a quantity

Accumulate the weighted mean of a given quantity.

Uses the numerically stable weighted mean algorithm described in ["Incremental calculation of weighted mean and variance," Tony Finch, _Univeristy of Cambridge Computing Service,_ 2009.](http://www-uxsup.csx.cam.ac.uk/~fanf2/hermes/doc/antiforgery/stats.pdf)

### Averaging constructor and required members

```python
Average.ing(quantity)
```

  * `quantity` (function returning double) computes the quantity of interest from the data.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `mean` (mutable double) is the running mean, initially 0.0. Note that this value contributes to the total mean with weight zero (because `entries` is initially zero), so this arbitrary choice does not bias the final result.

### Averaged constructor and required members

```python
Average.ed(entries, mean)
```

  * `entries` (double) is the number of entries.
  * `mean` (double) is the mean.

### Fill and combine algorithms

The relevant part of the `fill` method is the three lines in its `else` clause. Complications arise when handling infinite and NaN values such that `combine` is associative.

```python
def fill(averaging, datum, weight):
    if weight > 0.0:
        q = averaging.quantity(datum)
        averaging.entries += weight

        if math.isnan(averaging.mean) or math.isnan(q):
            averaging.mean = float("nan")

        elif math.isinf(averaging.mean) or math.isinf(q):
            if math.isinf(averaging.mean) and math.isinf(q) and averaging.mean * q < 0.0:
                averaging.mean = float("nan")  # opposite-sign infinities is bad
            elif math.isinf(q):
                averaging.mean = q             # mean becomes infinite with sign of q
            else:
                pass                           # mean is already infinite
            if math.isinf(averaging.entries) or math.isnan(averaging.entries):
                averaging.mean = float("nan")  # non-finite denominator is bad

        else:                                  # handle finite case
            delta = q - averaging.mean
            shift = delta * weight / averaging.entries
            averaging.mean += shift

def combine(one, two):
    entries = one.entries + two.entries
    if entries == 0.0:
        mean = (one.mean + two.mean) / 2.0
    else:
        mean = (one.entries*one.mean + two.entries*two.mean)/entries
    return Average.ed(entries, mean)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `mean` (JSON number, "nan", "inf", or "-inf")
  * optional `name` (JSON string), name of the `quantity` function, if provided.

**Example:**

```json
{"type": "Average",
 "data": {"entries": 123.0, "mean": 3.14, "name": "myfunc"}}
```

## **Deviate:** mean and variance

Accumulate the weighted mean and weighted variance of a given quantity.

The variance is computed around the mean, not zero.

Uses the numerically stable weighted mean and weighted variance algorithms described in ["Incremental calculation of weighted mean and variance," Tony Finch, _Univeristy of Cambridge Computing Service,_ 2009.](http://www-uxsup.csx.cam.ac.uk/~fanf2/hermes/doc/antiforgery/stats.pdf)

### Deviating constructor and required members

```python
Deviate.ing(quantity)
```

  * `quantity` (function returning double) computes the quantity of interest from the data.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `mean` (mutable double) is the running mean, initially 0.0. Note that this value contributes to the total mean with weight zero (because `entries` is initially zero), so this arbitrary choice does not bias the final result.
  * `variance` (mutable double) is the running variance, initially 0.0. Note that this also contributes nothing to the final result.

### Deviated constructor and required members

```python
Deviate.ed(entries, mean, variance)
```

  * `entries` (double) is the number of entries.
  * `mean` (double) is the mean.
  * `variance` (double) is the variance.

### Fill and combine algorithms

The relevant part of the `fill` method is the four lines in its `else` clause. Complications arise when handling infinite and NaN values such that `combine` is associative.

```python
def fill(deviating, datum, weight):
    if weight > 0.0:
        q = deviating.quantity(datum)
        varianceTimesEntries = deviating.variance * deviating.entries
        deviating.entries += weight

        if math.isnan(deviating.mean) or math.isnan(q):
            deviating.mean = float("nan")
            varianceTimesEntries = float("nan")

        elif math.isinf(deviating.mean) or math.isinf(q):
            if math.isinf(deviating.mean) and math.isinf(q) and deviating.mean * q < 0.0:
                deviating.mean = float("nan") # opposite-sign infinities is bad
            elif math.isinf(q):
                deviating.mean = q            # mean becomes infinite with sign of q
            else:
                pass                          # mean and variance are already infinite
            if math.isinf(deviating.entries) or math.isnan(deviating.entries):
                deviating.mean = float("nan") # non-finite denominator is bad

            # any infinite value makes the variance NaN
            varianceTimesEntries = float("nan")

        else:                                 # handle finite case
            delta = q - deviating.mean
            shift = delta * weight / deviating.entries
            deviating.mean += shift
            varianceTimesEntries += weight * delta * (q - deviating.mean)

        deviating.variance = varianceTimesEntries / deviating.entries

def combine(one, two):
    entries = one.entries + two.entries
    if entries == 0.0:
        mean = (one.mean + two.mean) / 2.0
    else:
        mean = (one.entries*one.mean + two.entries*two.mean) / entries

    varianceTimesEntries = one.entries*one.variance + two.entries*two.variance \
                           + one.entries*one.mean**2 + two.entries*two.mean**2 \
                           - 2.0*mean*(one.entries*one.mean + two.entries*two.mean) \
                           + entries*mean**2

    if entries == 0.0:
        variance = varianceTimesEntries
    else:
        variance = varianceTimesEntries / entries

    return Deviate.ed(entries, mean, variance)
```

**Note:** the `merge` function is not explicitly defined in Tony Finch's paper, but it's derivable from the algebra.

**FIXME:** the last two terms in `varianceTimesEntries` can be simplified. But now that we've added a special case for `mean` when `entries` is zero, which of the factors of `mean` and `(one.entries*one.mean + two.entries*two.mean)` should use the unweighted value?

This is only relevant for `Deviated` objects constructed by hand: `Deviating` objects aggregated in the normal way would _always_ have `mean` and `variance` in their initial state if `entries == 0.0` (because negative weights cannot contribute).

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `mean` (JSON number, "nan", "inf", or "-inf")
  * `variance` (JSON number, "nan", "inf", or "-inf")
  * optional `name` (JSON string), name of the `quantity` function, if provided.

**Example:**

```json
{"type": "Deviate",
 "data": {"entries": 123.0, "mean": 3.14, "variance": 0.1, "name": "myfunc"}}
```

## **Minimize:** minimum value

Find the minimum value of a given quantity. If no data are observed, the result is NaN.

### Minimizing constructor and required members

```python
Minimize.ing(quantity)
```

  * `quantity` (function returning double) computes the quantity of interest from the data.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `min` (mutable double) is the lowest value of the quantity observed, initially NaN.

### Minimized constructor and required members

```python
Minimize.ed(entries, min)
```

  * `entries` (double) is the number of entries.
  * `min` (double) is the lowest value of the quantity observed or NaN if no data were observed.

### Fill and combine algorithms

```python
def fill(minimizing, datum, weight):
    if weight > 0.0:
        q = minimizing.quantity(datum)
        minimizing.entries += weight
        if math.isnan(minimizing.min) or q < minimizing.min:
            minimizing.min = q

def combine(one, two):
    entries = one.entries + two.entries
    if math.isnan(one.min):
        min = two.min
    elif math.isnan(two.min):
        min = one.min
    elif one.min < two.min:
        min = one.min
    else:
        min = two.min
    return Minimize.ed(entries, min)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `min` (JSON number, "nan", "inf", or "-inf")
  * optional `name` (JSON string), name of the `quantity` function, if provided.

**Example:**

```json
{"type": "Minimize",
 "data": {"entries": 123.0, "min": 3.14, "name": "myfunc"}}
```

## **Maximize:** maximum value

Find the maximum value of a given quantity. If no data are observed, the result is NaN.

### Maximizing constructor and required members

```python
Maximize.ing(quantity)
```

  * `quantity` (function returning double) computes the quantity of interest from the data.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `max` (mutable double) is the highest value of the quantity observed, initially NaN.

### Maximized constructor and required members

```python
Maximize.ed(entries, min)
```

  * `entries` (double) is the number of entries.
  * `max` (double) is the highest value of the quantity observed or NaN if no data were observed.

### Fill and combine algorithms

```python
def fill(maximizing, datum, weight):
    if weight > 0.0:
        q = maximizing.quantity(datum)
        maximizing.entries += weight
        if math.isnan(maximizing.max) or q > maximizing.max:
            maximizing.max = q

def combine(one, two):
    entries = one.entries + two.entries
    if math.isnan(one.max):
        max = two.max
    elif math.isnan(two.max):
        max = one.max
    elif one.max > two.max:
        max = one.max
    else:
        max = two.max
    return Maximize.ed(entries, max)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `max` (JSON number, "nan", "inf", or "-inf")
  * optional `name` (JSON string), name of the `quantity` function, if provided.

**Example:**

```json
{"type": "Maximize",
 "data": {"entries": 123.0, "max": 3.14, "name": "myfunc"}}
```

## **Bag:** accumulate values for scatter plots

Accumulate raw numbers, vectors of numbers, or strings, with identical values merged.

A bag is the appropriate data type for scatter plots: a container that collects raw values, maintaining multiplicity but not order. (A "bag" is also known as a "multiset.") Conceptually, it is a mapping from distinct raw values to the number of observations: when two instances of the same raw value are observed, one key is stored and their weights add.

Although the user-defined function may return scalar numbers, fixed-dimension vectors of numbers, or categorical strings, it may not mix types. Different Bag primitives in an analysis tree may collect different types.

Consider using Bag with [Limit](#limit-keep-detail-until-entries-is-large) for collections that roll over to a mere count when they exceed a limit.

### Bagging constructor and required members

```python
Bag.ing(quantity)
```
  * `quantity` (function returning a double, a vector of doubles, or a string) computes the quantity of interest from the data.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `values` (mutable map from quantity return type to double) is the number of entries for each unique item.

### Bagged constructor and required members

```python
Bag.ed(entries, values)
```

  * `entries` (double) is the number of entries.
  * `values` (map from double, vector of doubles, or string to double) is the number of entries for each unique item.

### Fill and combine algorithms

```python
def fill(bagging, datum, weight):
    if weight > 0.0:
        q = bagging.quantity(datum)
        if math.isnan(q):   # something to avoid NaN != NaN
            q = "nan"       # (handling is more complex in type-safe languages)
        bagging.entries += weight
        if q in bagging.values:
            bagging.values[q] += weight
        else:
            bagging.values[q] = weight

def combine(one, two):
    entries = one.entries + two.entries
    values = {}
    for v in set(one.values.keys()).union(set(two.values.keys())):
        if v in one.values and v in two.values:
            values[v] = one.values[v] + two.values[v]
        elif v in one.values:
            values[v] = one.values[v]
        elif v in two.values:
            values[v] = two.values[v]
    return Bag.ed(entries, values)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `values` (JSON array of JSON objects containing `w` (JSON number), the total weight of entries for a unique value and `v` (JSON number, array of numbers, or string), the value); canonical form is sorted by `v` (lexicographically)
  * optional `name` (JSON string), name of the `quantity` function, if provided.

**Examples:**

```json
{"type": "Bag",
 "data": {
   "entries": 123.0,
   "values": [
     {"w": 23.0, "v": -999.0},
     {"w": 20.0, "v": -4.0},
     {"w": 20.0, "v": -2.0},
     {"w": 30.0, "v": 0.0},
     {"w": 30.0, "v": 2.0}]}}
```

```json
{"type": "Bag",
 "data": {
   "entries": 123.0,
   "values": [
     {"w": 23.0, "v": [1.0, 2.0, 3.0]},
     {"w": 20.0, "v": [3.14, 3.14, 3.14]},
     {"w": 20.0, "v": [99.0, 50.0, 1.0]},
     {"w": 30.0, "v": [7.0, 2.2, 9.8]},
     {"w": 30.0, "v": [33.3, 66.6, 99.9]}]}}
```

```json
{"type": "Bag",
 "data": {
   "entries": 123.0,
   "values": [
     {"w": 23.0, "v": "five"},
     {"w": 20.0, "v": "four"},
     {"w": 20.0, "v": "one"},
     {"w": 30.0, "v": "three"},
     {"w": 30.0, "v": "two"}]}}
```

# Second kind: pass to different sub-aggregators based on values seen in data

## **Bin:** regular binning for histograms

Split a quantity into equally spaced bins between a low and high threshold and fill exactly one bin per datum.

When composed with [Count](#count-sum-of-weights), this produces a standard histogram:

```python
Bin.ing(100, 0, 10, fill_x, Count.ing())
```

and when nested, it produces a two-dimensional histogram:

```python
Bin.ing(100, 0, 10, fill_x,
  Bin.ing(100, 0, 10, fill_y, Count.ing()))
```

Combining with [Deviate](#deviate-mean-and-variance) produces a physicist's "profile plot:"

```python
Bin.ing(100, 0, 10, fill_x, Deviate.ing(fill_y))
```

and so on.

### Binning constructor and required members

```python
Bin.ing(num, low, high, quantity, value=Count.ing(), underflow=Count.ing(), overflow=Count.ing(), nanflow=Count.ing())
```

  * `num` (32-bit integer) is the number of bins; must be at least one.
  * `low` (double) is the minimum-value edge of the first bin.
  * `high` (double) is the maximum-value edge of the last bin; must be strictly greater than `low`.
  * `quantity` (function returning double) computes the quantity of interest from the data.
  * `value` (present-tense aggregator) generates sub-aggregators to put in each bin.
  * `underflow` (present-tense aggregator) is a sub-aggregator to use for data whose quantity is less than `low`.
  * `overflow` (present-tense aggregator) is a sub-aggregator to use for data whose quantity is greater than or equal to `high`.
  * `nanflow` (present-tense aggregator) is a sub-aggregator to use for data whose quantity is NaN.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `values` (list of present-tense aggregators) are the sub-aggregators in each bin.

### Binned constructor and required members

```python
Bin.ed(low, high, entries, values, underflow, overflow, nanflow)
```

  * `low` (double) is the minimum-value edge of the first bin.
  * `high` (double) is the maximum-value edge of the last bin; must be strictly greater than `low`.
  * `entries` (double) is the number of entries.
  * `values` (list of past-tense aggregators) is the filled sub-aggregators, one for each bin.
  * `underflow` (past-tense aggregator) is the filled underflow bin.
  * `overflow` (past-tense aggregator) is the filled overflow bin.
  * `nanflow` (past-tense aggregator) is the filled nanflow bin.

### Fill and combine algorithms

```python
def fill(binning, datum, weight):
    if weight > 0.0:
        q = binning.quantity(datum)
        if math.isnan(q):
            fill(binning.nanflow, datum, weight)
        elif q < binning.low:
            fill(binning.underflow, datum, weight)
        elif q >= binning.high:
            fill(binning.overflow, datum, weight)
        else:
            bin = int(math.floor(binning.num * \
                (q - binning.low) / (binning.high - binning.low)))
            fill(binning.values[bin], datum, weight)
        binning.entries += weight

def combine(one, two):
    if one.num != two.num or one.low != two.low or one.high != two.high:
        raise Exception
    entries = one.entries + two.entries
    values = [combine(x, y) for x, y in zip(one.values, two.values)]
    underflow = combine(one.underflow, two.underflow)
    overflow = combine(one.overflow, two.overflow)
    nanflow = combine(one.nanflow, two.nanflow)
    return Bin.ed(one.low, one.high, entries, values, underflow, overflow, nanflow)
```

### JSON fragment format

JSON object containing

  * `low` (JSON number)
  * `high` (JSON number)
  * `entries` (JSON number or "inf")
  * `values:type` (JSON string), name of the values sub-aggregator type
  * `values` (JSON array of sub-aggregators)
  * `underflow:type` (JSON string), name of the underflow sub-aggregator type
  * `underflow` sub-aggregator
  * `overflow:type` (JSON string), name of the overflow sub-aggregator type
  * `overflow` sub-aggregator
  * `nanflow:type` (JSON string), name of the nanflow sub-aggregator type
  * `nanflow` (sub-aggregator)
  * optional `name` (JSON string), name of the `quantity` function, if provided.
  * optional `values:name` (JSON string), name of the `quantity` function used by each value. If specified here, it is _not_ specified in all the values, thereby streamlining the JSON.

**Examples:**

Here is a five-bin histogram, whose bin centers are at -4, -2, 0, 2, and 4. It counts the number of measurements made at each position.

```json
{"type": "Bin",
 "data": {
   "low": -5.0,
   "high": 5.0,
   "entries": 123.0,
   "name": "position [cm]",
   "values:type": "Count",
   "values": [10.0, 20.0, 20.0, 30.0, 30.0],
   "underflow:type": "Count",
   "underflow": 5.0,
   "overflow:type": "Count",
   "overflow": 8.0,
   "nanflow:type": "Count",
   "nanflow": 0.0}}
```

Here is another five-bin histogram on the same domain, this one quantifying an average value in each bin. The quantity measured by the average has a name (`"average time [s]"`), which would have been a `"name"` field in the JSON objects representing the averages if it had not been specified once in `"values:name"`.

```json
{"type": "Bin",
 "data": {
   "low": -5.0,
   "high": 5.0,
   "entries": 123.0,
   "name": "position [cm]",
   "values:type": "Average",
   "values:name": "average time [s]",
   "values": [
     {"entries": 10.0, "mean": 4.25},
     {"entries": 20.0, "mean": 16.21},
     {"entries": 20.0, "mean": 20.28},
     {"entries": 30.0, "mean": 16.19},
     {"entries": 30.0, "mean": 4.23}],
   "underflow:type": "Count",
   "underflow": 5.0,
   "overflow:type": "Count",
   "overflow": 8.0,
   "nanflow:type": "Count",
   "nanflow": 0.0}}
```

## **SparselyBin:** ignore zeros

Split a quantity into equally spaced bins, creating them whenever their `entries` would be non-zero. Exactly one sub-aggregator is filled per datum.

Use this when you have a distribution of known scale (bin width) but unknown domain (lowest and highest bin index).

Unlike fixed-domain binning, this aggregator has the potential to use unlimited memory. A large number of _distinct_ outliers can generate many unwanted bins.

Like fixed-domain binning, the bins are indexed by integers, though they are 64-bit and may be negative. Bin indexes below `-(2**63 - 1)` are put in the `-(2**63 - 1)` are bin and indexes above `(2**63 - 1)` are put in the `(2**63 - 1)` bin.

### SparselyBinning constructor and required members

```python
SparselyBin.ing(binWidth, quantity, value=Count.ing(), nanflow=Count.ing(), origin=0.0)
```

  * `binWidth` (double) is the width of a bin; must be strictly greater than zero.
  * `quantity` (function returning double) computes the quantity of interest from the data.
  * `value` (present-tense aggregator) generates sub-aggregators to put in each bin.
  * `nanflow` (present-tense aggregator) is a sub-aggregator to use for data whose quantity is NaN.
  * `origin` (double) is the left edge of the bin whose index is 0.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `bins` (mutable map from 64-bit integer to present-tense aggregator) is the map, probably a hashmap, to fill with values when their `entries` become non-zero.

### SparselyBinned constructor and required members

```python
SparselyBin.ed(binWidth, entries, contentType, bins, nanflow, origin)
```

  * `binWidth` (double) is the width of a bin.
  * `entries` (double) is the number of entries.
  * `contentType` (string) is the value's sub-aggregator type (must be provided to determine type for the case when `bins` is empty).
  * `bins` (map from 64-bit integer to past-tense aggregator) is the non-empty bin indexes and their values.
  * `nanflow` (past-tense aggregator) is the filled nanflow bin.
  * `origin` (double) is the left edge of the bin whose index is zero.

### Fill and combine algorithms

```python
def fill(sparselybinning, datum, weight):
    if weight > 0.0:
        q = sparselybinning.quantity(datum)
        if math.isnan(q):
            fill(sparselybinning.nanflow, datum, weight)
        else:
            softbin = (q - sparselybinning.origin) / sparselybinning.binWidth

            if softbin <= -(2**63 - 1):
                bin = -(2**63 - 1)
            elif softbin >= (2**63 - 1):
                bin = (2**63 - 1)
            else:
                bin = int(math.floor(softbin))

            if bin not in sparselybinning.bins:
                sparselybinning.bins[bin] = sparselybinning.value.copy()

            fill(sparselybinning.bins[bin], datum, weight)

        sparselybinning.entries += weight

def combine(one, two):
    if one.binWidth != two.binWidth or one.origin != two.origin or one.contentType != two.contentType:
        raise Exception
    entries = one.entries + two.entries
    contentType = one.contentType
    bins = {}
    for key in set(one.bins.keys()).union(set(two.bins.keys())):
        if key in one.bins and key in two.bins:
            bins[key] = combine(one.bins[key], two.bins[key])
        elif key in one.bins:
            bins[key] = one.bins[key].copy()
        elif key in two.bins:
            bins[key] = two.bins[key].copy()
    nanflow = combine(one.nanflow, two.nanflow)
    return SparselyBin.ed(one.binWidth, entries, contentType, \
                          bins, nanflow, one.origin)
```

### JSON fragment format

JSON object containing

  * `binWidth` (JSON number)
  * `entries` (JSON number or "inf")
  * `bins:type` (JSON string), name of the bins sub-aggregator type
  * `bins` (JSON object), keys are string representations of the bin indexes (decimal, no leading zeros) and values are sub-aggregators
  * `nanflow:type` (JSON string), name of the nanflow sub-aggregator type
  * `nanflow` (sub-aggregator)
  * `origin` (JSON number)
  * optional `name` (JSON string), name of the `quantity` function, if provided.
  * optional `values:name` (JSON string), name of the `quantity` function used by each bin value. If specified here, it is _not_ specified in all the values, thereby streamlining the JSON.

**Example:**

```json
{"type": "SparselyBin",
 "data": {
   "binWidth": 2.0,
   "entries": 123.0,
   "bins:type": "Count",
   "bins": {"-999": 5.0, "-4": 10.0, "-2": 20.0, "0": 20.0, "2": 30.0, "4": 30.0, "12345": 8.0},
   "nanflow:type": "Count",
   "nanflow": 0.0,
   "origin": 0.0,
   "name": "myfunc"}}
```

## **CentrallyBin:** fully partitioning with centers

Split a quantity into bins defined by irregularly spaced bin centers, with exactly one sub-aggregator filled per datum (the closest one).

Unlike irregular bins defined by low-high ranges, irregular bins defined by bin centers are guaranteed to fully partition the space with no gaps and no overlaps. It could be viewed as cluster scoring in one dimension.

### CentrallyBinning constructor and required members

```python
CentrallyBin.ing(centers, quantity, value=Count.ing(), nanflow=Count.ing())
```

  * `centers` (list of doubles) is the centers of all bins
  * `quantity` (function returning double) computes the quantity of interest from the data.
  * `value` (present-tense aggregator) generates sub-aggregators to put in each bin.
  * `nanflow` (present-tense aggregator) is a sub-aggregator to use for data whose quantity is NaN.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `bins` (list of double, present-tense aggregator pairs) are the bin centers and sub-aggregators in each bin.

### CentrallyBinned constructor and required members

```python
CentrallyBin.ed(entries, bins, nanflow)
```

  * `entries` (double) is the number of entries.
  * `bins` (list of double, past-tense aggregator pairs) is the list of bin centers and their accumulated data.
  * `nanflow` (past-tense aggregator) is the filled nanflow bin.

### Fill and combine algorithms

```python
def fill(centrallybinning, datum, weight):
    if weight > 0.0:
        q = centrallybinning.quantity(datum)
        if math.isnan(q):
            fill(centrallybinning.nanflow, datum, weight)
        else:
            if math.isinf(q):
                if q < 0.0:
                    closest = centrallybinning.bins[0][1]
                else:
                    closest = centrallybinning.bins[-1][1]
            else:
                dist, closest = min((abs(c - q), v) for c, v in centrallybinning.bins)
            fill(closest, datum, weight)
        centrallybinning.entries += weight

def combine(one, two):
    if set(one.centers) != set(two.centers):
        raise Exception
    entries = one.entries + two.entries
    bins = []
    for c1, v1 in one.bins:
        v2 = [v for c2, v in two.bins if c1 == c2][0]
        bins.append((c1, combine(v1, v2)))

    nanflow = combine(one.nanflow, two.nanflow)
    return CentrallyBin.ed(entries, bins, nanflow)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `bins:type` (JSON string), name of the bins sub-aggregator type
  * `bins` (JSON array of JSON objects containing `center` (JSON number) and `value` (sub-aggregator)), collection of bin centers and their associated data
  * `nanflow:type` (JSON string), name of the nanflow sub-aggregator type
  * `nanflow` (sub-aggregator)
  * optional `name` (JSON string), name of the `quantity` function, if provided.
  * optional `bins:name` (JSON string), name of the `quantity` function used by each bin value. If specified here, it is _not_ specified in all the values, thereby streamlining the JSON.

**Examples:**

```json
{"type": "CentrallyBin",
 "data": {
   "entries": 123.0,
   "bins:type": "Count",
   "bins": [
     {"center": -999.0, "value": 5.0},
     {"center": -4.0, "value": 10.0},
     {"center": -2.0, "value": 20.0},
     {"center": 0.0, "value": 20.0},
     {"center": 2.0, "value": 30.0},
     {"center": 4.0, "value": 30.0},
     {"center": 12345.0, "value": 8.0}],
   "nanflow:type": "Count",
   "nanflow": 0.0,
   "name": "myfunc"}}
```

Here is an example with `Average` sub-aggregators:

```json
{"type": "CentrallyBin",
 "data": {
   "entries": 123.0,
   "bins:type": "Average",
   "bins": [
     {"center": -999.0, "value": 5.0, "mean": -1.0},
     {"center": -4.0, "value": 10.0, "mean": 4.25},
     {"center": -2.0, "value": 20.0, "mean": 16.21},
     {"center": 0.0, "value": 20.0, "mean": 20.28},
     {"center": 2.0, "value": 20.0, "mean": 16.19},
     {"center": 4.0, "value": 30.0, "mean": 4.23},
     {"center": 12345.0, "value": 8.0, "mean": -1.0}],
   "nanflow:type": "Count",
   "nanflow": 0.0,
   "name": "myfunc",
   "bins:name": "myfunc2"}}
```

## **IrregularlyBin:** fully partitioning with edges

Accumulate a suite of aggregators, each between two thresholds, filling exactly one per datum.

This is a variation on [Stack](#stack-cumulative-filling), which fills `N + 1` aggregators with `N` successively tighter cut thresholds. IrregularlyBin fills `N + 1` aggregators in the non-overlapping intervals between `N` thresholds.

IrregularlyBin is also similar to [CentrallyBin](#centrallybin-fully-partitioning-with-centers), in that they both partition a space into irregular subdomains with no gaps and no overlaps. However, CentrallyBin is defined by bin centers and IrregularlyBin is defined by bin low edges, the first of which is negative infinity (inclusive) and the last is implicitly positive infinity (inclusive).

### IrregularlyBinning constructor and required members

```python
IrregularlyBin.ing(thresholds, quantity, value, nanflow)
```

  * `thresholds` (list of doubles) specifies `N` lower cut thresholds, so the IrregularlyBin will fill `N + 1` aggregators in distinct intervals.
  * `quantity` (function returning double) computes the quantity of interest from the data.
  * `value` (present-tense aggregator) generates sub-aggregators for each bin.
  * `nanflow` (present-tense aggregator) is a sub-aggregator to use for data whose quantity is NaN.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `bins` (list of double, present-tense aggregator pairs) are the `N + 1` lower thresholds and sub-aggregators. The first threshold is minus infinity; the rest are the ones specified by `thresholds`.

### IrregularlyBinned constructor and required members

```python
IrregularlyBin.ed(entries, bins, nanflow)
```

  * `entries` (double) is the number of entries.
  * `bins` (list of double, past-tense aggregator pairs) are the `N + 1` thresholds and sub-aggregator pairs.
  * `nanflow` (past-tense aggregator) is the filled nanflow bin.

### Fill and combine algorithms

```python
def fill(irregularlybinning, datum, weight):
    if weight > 0.0:
        q = irregularlybinning.quantity(datum)
        if math.isnan(q):
            fill(irregularlybinning.nanflow, datum, weight)
        else:
            lowEdges = irregularlybinning.bins
            highEdges = list(irregularlybinning.bins[1:]) + [(float("nan"), None)]
            for (low, sub), (high, _) in zip(lowEdges, highEdges):
                if q >= low and not q >= high:    # include high endpoint only for the last bin
                    fill(sub, datum, weight)
                    break
        irregularlybinning.entries += weight

def combine(one, two):
    if one.thresholds != two.thresholds:
        raise Exception
    entries = one.entries + two.entries
    bins = [(c, combine(v1, v2)) for (c, v1), (_, v2) in zip(one.bins, two.bins)]
    nanflow = combine(one.nanflow, two.nanflow)
    return IrregularlyBin.ed(entries, bins, nanflow)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `type` (JSON string), name of the sub-aggregator type
  * `data` (JSON array of JSON objects containing `atleast` (JSON number or "-inf") and `data` (sub-aggregator)), collection of lower cut thresholds (including minus infinity) and their associated data
  * `nanflow:type` (JSON string), name of the nanflow sub-aggregator type
  * `nanflow` (sub-aggregator)
  * optional `name` (JSON string), name of the `quantity` function, if provided.
  * optional `data:name` (JSON string), name of the `quantity` function used by the sub-aggregators. If specified here, it is _not_ specified in all the sub-aggregators, thereby streamlining the JSON.

**Examples:**

```json
{"type": "IrregularlyBin",
 "data": {
   "entries": 123.0,
   "type": "Count",
   "data": [
     {"atleast": "-inf", "data": 23.0},
     {"atleast": 1.0, "data": 20.0},
     {"atleast": 2.0, "data": 20.0},
     {"atleast": 3.0, "data": 30.0},
     {"atleast": 4.0, "data": 30.0}],
   "nanflow:type": "Count",
   "nanflow": 0.0,
   "name": "myfunc"}}
```

```json
{"type": "IrregularlyBin",
 "data": {
   "entries": 123.0,
   "type": "Average",
   "data": [
     {"atleast": "-inf", "data": {"entries": 23.0, "mean": 3.14}},
     {"atleast": 1.0, "data": {"entries": 20.0, "mean": 2.28}},
     {"atleast": 2.0, "data": {"entries": 20.0, "mean": 1.16}},
     {"atleast": 3.0, "data": {"entries": 30.0, "mean": 8.9}},
     {"atleast": 4.0, "data": {"entries": 30.0, "mean": 22.7}}],
   "nanflow:type": "Count",
   "nanflow": 0.0}}
```

## **Categorize:** string-valued bins, bar charts

Split a given quantity by its categorical value and fill only one category per datum.

A bar chart may be thought of as a histogram with string-valued (categorical) bins, so this is the equivalent of [Bin](#bin-regular-binning-for-histograms) for bar charts. The order of the strings is deferred to the visualization stage.

Unlike [SparselyBin](#sparselybin-ignore-zeros), this aggregator has the potential to use unlimited memory. A large number of _distinct_ categories can generate many unwanted bins.

### Categorizing constructor and required members

```python
Categorize.ing(quantity, value=Count.ing())
```

  * `quantity` (function returning double) computes the quantity of interest from the data.
  * `value` (present-tense aggregator) generates sub-aggregators to put in each bin.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `pairs` (mutable map from string to present-tense aggregator) is the map, probably a hashmap, to fill with values when their `entries` become non-zero.

### Categorized constructor and required members

```python
Categorize.ed(entries, contentType, pairs)
```

  * `entries` (double) is the number of entries.
  * `contentType` (string) is the value's sub-aggregator type (must be provided to determine type for the case when `bins` is empty).
  * `pairs` (map from string to past-tense aggregator) is the non-empty bin categories and their values.

### Fill and combine algorithms

```python
def fill(categorizing, datum, weight):
    if weight > 0.0:
        q = categorizing.quantity(datum)
        if q not in categorizing.pairs:
            categorizing.pairs[q] = categorizing.value.copy()
        fill(categorizing.pairs[q], datum, weight)
        categorizing.entries += weight

def combine(one, two):
    if one.contentType != two.contentType:
        raise Exception
    entries = one.entries + two.entries
    contentType = one.contentType
    pairs = {}
    for key in set(one.pairs.keys()).union(set(two.pairs.keys())):
        if key in one.pairs and key in two.pairs:
            pairs[key] = combine(one.pairs[key], two.pairs[key])
        elif key in one.pairs:
            pairs[key] = one.pairs[key].copy()
        elif key in two.pairs:
            pairs[key] = two.pairs[key].copy()
    return Categorize.ed(entries, contentType, pairs)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `type` (JSON string), name of the sub-aggregator type
  * `data` (JSON object), keys are the string-valued categories and values are sub-aggregators
  * optional `name` (JSON string), name of the `quantity` function, if provided.
  * optional `bins:name` (JSON string), name of the `quantity` function used by each bin value. If specified here, it is _not_ specified in all the values, thereby streamlining the JSON.

**Example:**

```json
{"type": "Categorize",
 "data": {
   "entries": 123.0,
   "type": "Count",
   "data": {"one": 23.0, "two": 20.0, "three": 20.0, "four": 30.0, "five": 30.0},
   "name": "myfunc"}}
```

## **Fraction:** efficiency plots

Accumulate two aggregators, one containing only entries that pass a given selection (numerator) and another that contains all entries (denominator).

The aggregator may be a simple [Count](#count-sum-of-weights) to measure the efficiency of a cut, a [Bin](#bin-regular-binning-for-histograms) to plot a turn-on curve, or anything else to be tested with and without a cut.

As a side effect of NaN values returning false for any comparison, a NaN return value from the selection is treated as a failed cut (the denominator is filled but the numerator is not).

Note that the selection function can return any non-negative number. Fractional selections would fractionally weight the numerator and would not affect the denominator. It is even possible to make the numerator exceed the denominator with weights greater than 1.0.

### Fractioning constructor and required members

```python
Fraction.ing(quantity, value=Count.ing())
```

  * `quantity` (function returning boolean or double) computes the quantity of interest from the data and interprets it as a selection (multiplicative factor on weight).
  * `value` (present-tense aggregator) generates sub-aggregators for the numerator and denominator.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `numerator` (present-tense aggregator) is the sub-aggregator of entries that pass the selection.
  * `denominator` (present-tense aggregator) is the sub-aggregator of all entries.

### Fractioned constructor and required members

```python
Fraction.ed(entries, numerator, denominator)
```

  * `entries` (double) is the number of entries.
  * `numerator` (past-tense aggregator) is the filled numerator.
  * `denominator` (past-tense aggregator) is the filled denominator.

### Fractioned alternate constructor

```python
Fraction.build(numerator, denominator)
```

  * `numerator` (past-tense aggregator) is the filled numerator.
  * `denominator` (past-tense aggregator) is the filled denominator.

This constructor will make a past-tense Fraction object that can be used to represent any ratio, not just a fraction. The numerator and denominator may come from different sources, but they must have compatible binning.

### Fill, combine, and alternate constructor algorithms

```python
def fill(fractioning, datum, weight):
    if weight > 0.0:
        w = weight * fractioning.quantity(datum)
        fill(fractioning.denominator, datum, weight)
        if w > 0.0:
            fill(fractioning.numerator, datum, w)
        fractioning.entries += weight

def combine(one, two):
    entries = one.entries + two.entries
    numerator = combine(one.numerator, two.numerator)
    denominator = combine(one.denominator, two.denominator)
    return Fraction.ed(entries, numerator, denominator)

def build(numerator, denominator):
    return Fraction.ed(denominator.entries, numerator, denominator)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `type` (JSON string), name of the numerator/denominator type
  * `numerator` (sub-aggregator)
  * `denominator` (sub-aggregator)
  * optional `name` (JSON string), name of the `quantity` function, if provided.
  * optional `sub:name` (JSON string), name of the `quantity` function used by the numerator and denominator. If specified here, it is _not_ specified in all the sub-aggregators, thereby streamlining the JSON.

**Example:**

```json
{"type": "Fraction",
 "data": {
   "entries": 123.0,
   "name": "trigger",
   "sub:name": "energy [GeV]",
   "type": "Bin",
   "numerator": {
     "low": -5.0,
     "high": 5.0,
     "entries": 98.0,
     "values:type": "Count",
     "values": [2.0, 15.0, 18.0, 25.0, 30.0],
     "underflow:type": "Count",
     "underflow": 0.0,
     "overflow:type": "Count",
     "overflow": 8.0,
     "nanflow:type": "Count",
     "nanflow": 0.0},
   "denominator": {
     "low": -5.0,
     "high": 5.0,
     "entries": 123.0,
     "values:type": "Count",
     "values": [10.0, 20.0, 20.0, 30.0, 30.0],
     "underflow:type": "Count",
     "underflow": 5.0,
     "overflow:type": "Count",
     "overflow": 8.0,
     "nanflow:type": "Count",
     "nanflow": 0.0}}}
```

## **Stack:** cumulative filling

Accumulates a suite of aggregators, each filtered with a tighter selection on the same quantity.

This is a generalization of [Fraction](#fraction-efficiency-plots), which fills two aggregators, one with a cut, the other without. Stack fills `N + 1` aggregators with `N` successively tighter cut thresholds. The first is always filled (like the denominator of Fraction), the second is filled if the computed quantity exceeds its threshold, the next is filled if the computed quantity exceeds a higher threshold, and so on.

The thresholds are presented in increasing order and the computed value must be greater than or equal to a threshold to fill the corresponding bin, and therefore the number of entries in each filled bin is greatest in the first and least in the last.

Although this aggregation could be plotted as a stack of histograms, stacks of histograms are often used to represent something different: data from different sources, rather than cuts on the same source. For example, it is common to stack distributions from different Monte Carlo simulations to show that they add up to the observed data. The Stack aggregator does not make plots of this type because aggregation trees in Histogrammar draw data from exactly one source.

To make plots from different sources in Histogrammar, one must perform separate aggregation runs. It may then be convenient to stack the results of those runs as though they were created with a Stack aggregation, so that plotting code can treat both cases uniformly. For this reason, Stack has an alternate constructor to build a Stack manually from distinct aggregators, even if those aggregators came from different aggregation runs.

### Stacking constructor and required members

```python
Stack.ing(thresholds, quantity, value, nanflow)
```

  * `thresholds` (list of doubles) specifies `N` lower cut thresholds, so the Stack will fill `N + 1` aggregators, each overlapping the last.
  * `quantity` (function returning double) computes the quantity of interest from the data.
  * `value` (present-tense aggregator) generates sub-aggregators for each bin.
  * `nanflow` (present-tense aggregator) is a sub-aggregator to use for data whose quantity is NaN.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `bins` (list of double, present-tense aggregator pairs) are the `N + 1` lower thresholds and sub-aggregators. The first is minus infinity; the rest are the ones specified by `thresholds`.

### Stacked constructor and required members

```python
Stack.ed(entries, bins, nanflow)
```

  * `entries` (double) is the number of entries.
  * `bins` (list of double, past-tense aggregator pairs) are the `N + 1` thresholds and sub-aggregator pairs.
  * `nanflow` (past-tense aggregator) is the filled nanflow bin.

### Stacked alternate constructor

```python
Stack.build(aggregators)
```

  * `aggregators` (list of aggregators of the same type from any source); the algorithm will attempt to add them, so they must also have the same binning/bounds/etc.

This constructor will make a past-tense Stacked object with NaN as cut thresholds and `Count.ed(0.0)` as `nanflow`.

### Fill, combine, and alternate constructor algorithms

```python
def fill(stacking, datum, weight):
    if weight > 0.0:
        q = stacking.quantity(datum)
        if math.isnan(q):
            fill(stacking.nanflow, datum, weight)
        else:
            for threshold, sub in stacking.bins:
                if q >= threshold:
                    fill(sub, datum, weight)
        stacking.entries += weight

def combine(one, two):
    if [c for c, v in one.bins] != [c for c, v in two.bins]:
        raise Exception
    entries = one.entries + two.entries
    bins = []
    for (c, v1), (_, v2) in zip(one.bins, two.bins):
        bins.append((c, combine(v1, v2)))
    nanflow = combine(one.nanflow, two.nanflow)
    return Stack.ed(entries, bins, nanflow)

def build(aggregators):
    entries = sum(x.entries for x in aggregators)
    bins = []
    for i in range(len(aggregators)):
        stackedAggregators = reduce(lambda a, b: combine(a, b), aggregators[i:])
        bins.append((float("nan"), stackedAggregators))
    return Stack.ed(entries, bins, Count.ed(0.0))
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `type` (JSON string), name of the sub-aggregator type
  * `data` (JSON array of JSON objects containing `atleast` (JSON number or "-inf") and `data` (sub-aggregator)), collection of lower cut thresholds (including minus infinity) and their associated data
  * `nanflow:type` (JSON string), name of the nanflow sub-aggregator type
  * `nanflow` (sub-aggregator)
  * optional `name` (JSON string), name of the `quantity` function, if provided.
  * optional `data:name` (JSON string), name of the `quantity` function used by the sub-aggregators. If specified here, it is _not_ specified in all the sub-aggregators, thereby streamlining the JSON.

**Examples:**

```json
{"type": "Stack",
 "data": {
   "entries": 123.0,
   "type": "Count",
   "data": [
     {"atleast": "-inf", "data": 123.0},
     {"atleast": 1.0, "data": 100.0},
     {"atleast": 2.0, "data": 82.0},
     {"atleast": 3.0, "data": 37.0},
     {"atleast": 4.0, "data": 4.0}],
   "nanflow:type": "Count",
   "nanflow": 0.0,
   "name": "myfunc"}}
```

```json
{"type": "Stack",
 "data": {
   "entries": 123.0,
   "type": "Average",
   "data": [
     {"atleast": "-inf", "data": {"entries": 123.0, "mean": 3.14}},
     {"atleast": 1.0, "data": {"entries": 100.0, "mean": 2.28}},
     {"atleast": 2.0, "data": {"entries": 82.0, "mean": 1.16}},
     {"atleast": 3.0, "data": {"entries": 37.0, "mean": 8.9}},
     {"atleast": 4.0, "data": {"entries": 4.0, "mean": 22.7}}],
   "nanflow:type": "Count",
   "nanflow": 0.0}}
```

## **Select:** apply a cut

Filter or weight data according to a given selection.

This primitive is a basic building block, intended to be used in conjunction with anything that needs a user-defined cut. In particular, a standard histogram often has a custom selection, and this can be built by nesting Select &rarr; Bin &rarr; Count.

Select also resembles [Fraction](#fraction-efficiency-plots), but without the `denominator`.

The efficiency of a cut in a Select aggregator named `x` is simply `x.cut.entries / x.entries` (because all aggregators have an `entries` member).

Note that the selection function can return any non-negative number. Fractional selections would fractionally weight the sub-aggregator.

### Selecting constructor and required members

```python
Select.ing(quantity, cut)
```

  * `quantity` (function returning boolean or double) computes the quantity of interest from the data and interprets it as a selection (multiplicative factor on weight).
  * `cut` (present-tense aggregator) will only be filled with data that pass the cut, and which are weighted by the cut.
  * `entries` (mutable double) is the number of entries, initially 0.0.

### Selected constructor and required members

```python
Select.ed(entries, cut)
```

  * `entries` (double) is the number of entries.
  * `cut` (past-tense aggregator) is the filled sub-aggregator.

### Fill and combine algorithms

```python
def fill(selecting, datum, weight):
    if weight > 0.0:
        w = weight * selecting.quantity(datum)
        if w > 0.0:
            fill(selecting.cut, datum, w)
        selecting.entries += weight

def combine(one, two):
    entries = one.entries + two.entries
    cut = combine(one.cut, two.cut)
    return Select.ed(entries, cut)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `type` (JSON string), name of the sub-aggregator type
  * `data` (sub-aggregator)
  * optional `name` (JSON string), name of the `quantity` function, if provided.

**Examples:**

```json
{"type": "Select",
 "data": {
   "entries": 123.0,
   "name": "trigger",
   "type": "Count",
   "data": 98.0}}
```

```json
{"type": "Select",
 "data": {
   "entries": 123.0,
   "name": "trigger",
   "sub:name": "energy [GeV]",
   "type": "Bin",
   "data": {
     "low": -5.0,
     "high": 5.0,
     "entries": 98.0,
     "values:type": "Count",
     "values": [2.0, 15.0, 18.0, 25.0, 30.0],
     "underflow:type": "Count",
     "underflow": 0.0,
     "overflow:type": "Count",
     "overflow": 8.0,
     "nanflow:type": "Count",
     "nanflow": 0.0}}}
```

## **Limit:** keep detail until entries is large

Accumulate an aggregator until its number of entries reaches a predefined limit.

Limit is intended to roll high-detail descriptions of small datasets over into low-detail descriptions of large datasets. For instance, a scatter plot is useful for small numbers of data points and heatmaps are useful for large ones. The following construction

```python
Bin(xbins, xlow, xhigh, lambda d: d.x,
  Bin(ybins, ylow, yhigh, lambda d: d.y,
    Limit(10.0, Bag(lambda d: [d.x, d.y]))))
```

fills a scatter plot in all x-y bins that have fewer than 10 entries and only a number of entries above that. Postprocessing code would use the bin-by-bin numbers of entries to color a heatmap and the raw data points to show outliers in the nearly empty bins.

Limit can effectively swap between two descriptions if it is embedded in a collection, such as [Branch](#branch-tuple-of-different-types). All elements of the collection would be filled until the Limit saturates, leaving only the low-detail one. For instance, one could aggregate several [SparselyBin](#sparselybin-ignore-zeros) histograms, each with a different `binWidth`, and progressively eliminate them in order of increasing `binWidth`.

Note that Limit saturates when it reaches a specified _total weight,_ not the number of data points in a [Bag](#bag-accumulate-values-for-scatter-plots), so it is not guaranteed to control memory use. (Imagine a dataset with many small weights.) However, the total weight is a more useful metric in data analysis.

### Limiting constructor and required members

```python
Limit.ing(limit, value)
```

  * `limit` (double) is the maximum number of entries (inclusive) before deleting the `value`.
  * `value` (present-tense aggregator) will only be filled until its number of entries exceeds the `limit`.
  * `entries` (mutable double) is the number of entries, initially 0.0.
  * `contentType` (string) is the value's sub-aggregator type (must be provided to determine type for the case when `value` has been deleted).

### Limited constructor and required members

```python
Limit.ed(entries, limit, contentType, value)
```

  * `entries` (double) is the number of entries.
  * `limit` (double) is the maximum number of entries (inclusive).
  * `contentType` (string) is the value's sub-aggregator type (must be provided to determine type for the case when `value` has been deleted).
  * `value` (past-tense aggregator or `None`) is the filled sub-aggregator if unsaturated, `None` if saturated.

### Fill and combine algorithms

```python
def fill(limiting, datum, weight):
    if weight > 0.0:
        if limiting.entries + weight > limiting.limit:
            limiting.value = None
        else:
            fill(limiting.value, datum, weight)
        limiting.entries += weight

def combine(one, two):
    if one.limit != two.limit or one.contentType != two.contentType:
        raise Exception
    entries = one.entries + two.entries
    if entries > one.limit:
        value = None
    else:
        value = combine(one.value, two.value)
    return Limit.ed(entries, one.limit, one.contentType, value)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `limit` (JSON number)
  * `type` (JSON string), name of the sub-aggregator type
  * `data` (sub-aggregator or JSON null)

**Examples:**

```json
{"type": "Limit",
 "data": {
   "entries": 98.0,
   "limit": 100.0,
   "type": "Bag",
   "data": {
     "entries": 98.0,
     "values": [
       {"w": 2.0, "v": [1.0, 2.0, 3.0]},
       {"w": 15.0, "v": [3.14, 3.14, 3.14]},
       {"w": 18.0, "v": [99.0, 50.0, 1.0]},
       {"w": 25.0, "v": [7.0, 2.2, 9.8]},
       {"w": 30.0, "v": [33.3, 66.6, 99.9]}]}}}
```

```json
{"type": "Limit",
 "data": {
   "entries": 123.0,
   "limit": 100.0,
   "type": "Bag",
   "data": null}}
```

# Third kind: broadcast to every sub-aggregator, independent of data

## **Label:** directory with string-based keys

Accumulate any number of aggregators of the same type and label them with strings. Every sub-aggregator is filled with every input datum.

This primitive simulates a directory of aggregators. For sub-directories, nest collections within the Label collection.

Note that all sub-aggregators within a Label must have the _same type_ (e.g. histograms of different binnings, but all histograms). To collect objects of _different types_ with string-based look-up keys, use [UntypedLabel](#untypedlabel-directory-of-different-types).

To collect aggregators of the _same type_ without naming them, use [Index](#index-list-with-integer-keys). To collect aggregators of _different types_ without naming them, use [Branch](#branch-tuple-of-different-types).

In strongly typed languages, the restriction to a single type allows nested objects to be extracted without casting.

### Labeling constructor and required members

```python
Label.ing(pairs)
```

  * `pairs` (map of string, present-tense aggregator pairs) is the collection of aggregators to fill.
  * `entries` (mutable double) is the number of entries, initially 0.0.

### Labeled constructor and required members

```python
Label.ed(entries, pairs)
```

  * `entries` (double) is the number of entries.
  * `pairs` (map of string, past-tense aggregator pairs) is the collection of filled aggregators.

### Fill and combine algorithms

```python
def fill(labeling, datum, weight):
    if weight > 0.0:
        for _, v in labeling.pairs.items():
            fill(v, datum, weight)
        labeling.entries += weight

def combine(one, two):
    if set(one.pairs.keys()) != set(two.pairs.keys()):
        raise Exception
    entries = one.entries + two.entries
    pairs = {}
    for l, v1 in one.pairs.items():
        v2 = two.pairs[l]
        pairs[l] = combine(v1, v2)
    return Label.ed(entries, pairs)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `type` (JSON string), name of the sub-aggregator type
  * `data` (JSON object), keys are the labels and values are sub-aggregators

The fact that Label requires all contents to have a single type allows the `type` to be specified once here, instead of once for every item in the collection. [UntypedLabel](#untypedlabel-directory-of-different-types) is more verbose.

**Example:**

```json
{"type": "Label",
 "data": {
   "entries": 123.0,
   "type": "Average",
   "data": {
     "one": {"entries": 123.0, "mean": 3.14},
     "two": {"entries": 123.0, "mean": 6.28},
     "three": {"entries": 123.0, "mean": 99.9}}}}
```

## **UntypedLabel:** directory of different types

Accumulate any number of aggregators of any type and label them with strings. Every sub-aggregator is filled with every input datum.

This primitive simulates a directory of aggregators. For sub-directories, nest collections within the UntypedLabel.

Note that sub-aggregators within an UntypedLabel may have _different types_. In strongly typed languages, this flexibility poses a problem: nested objects must be type-cast before they can be used. To collect objects of the _same type_ with string-based look-up keys, use [Label](#label-directory-with-string-based-keys).

To collect aggregators of the _same type_ without naming them, use [Index](#index-list-with-integer-keys). To collect aggregators of _different types_ without naming them, use [Branch](#branch-tuple-of-different-types).

### UntypedLabeling constructor and required members

```python
UntypedLabel.ing(pairs)
```

  * `pairs` (map of string, present-tense aggregator pairs) is the collection of aggregators to fill.
  * `entries` (mutable double) is the number of entries, initially 0.0.

### UntypedLabeled constructor and required members

```python
UntypedLabel.ed(entries, pairs)
```

  * `entries` (double) is the number of entries.
  * `pairs` (map of string, past-tense aggregator pairs) is the collection of filled aggregators.

### Fill and combine algorithms

```python
def fill(untypedlabeling, datum, weight):
    if weight > 0.0:
        for _, v in untypedlabeling.pairs.items():
            fill(v, datum, weight)
        untypedlabeling.entries += weight

def combine(one, two):
    if set(one.pairs.keys()) != set(two.pairs.keys()):
        raise Exception
    entries = one.entries + two.entries
    pairs = {}
    for l, v1 in one.pairs.items():
        v2 = two.pairs[l]
        pairs[l] = combine(v1, v2)
    return UntypedLabel.ed(entries, pairs)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `data` (JSON object mapping labels to JSON objects containing `type` (JSON string), name of sub-aggregator type and `data` (sub-aggregator))

The fact that UntypedLabel allows each element to have a different type forces the `type` annotation to be nested with the sub-aggregator. For collections that actually contain only one type, [Label](#label-directory-with-string-based-keys) avoids this redundancy.

**Example:**

```json
{"type": "UntypedLabel",
 "data": {
   "entries": 123.0,
   "data": {
     "one": {"type": "Count", "data": 123.0},
     "two": {"type": "Average", "data": {
       "entries": 123.0,
       "mean": 3.14}},
     "three": {"type": "Bin", "data": {
       "low": -5.0,
       "high": 5.0,
       "entries": 123.0,
       "name": "position [cm]",
       "values:type": "Count",
       "values": [10.0, 20.0, 20.0, 30.0, 30.0],
       "underflow:type": "Count",
       "underflow": 5.0,
       "overflow:type": "Count",
       "overflow": 8.0,
       "nanflow:type": "Count",
       "nanflow": 0.0}}}}}
```

## **Index:** list with integer keys

Accumulate any number of aggregators of the same type in a list. Every sub-aggregator is filled with every input datum.

This primitive provides an anonymous collection of aggregators (unless the integer index is taken to have special meaning, but generally such bookkeeping should be encoded in strings). Indexes can be nested to create two-dimensional ordinal grids of aggregators. (Use [Bin](#bin-regular-binning-for-histograms) if the space is to have a metric interpretation.)

Note that all sub-aggregators within an Index must have the _same type_ (e.g. histograms of different binnings, but all histograms). To collect objects of _different types,_ still indexed by integer, use [Branch](#branch-tuple-of-different-types).

To collect aggregators of the _same type_ with string-based labels, use [Label](#label-directory-with-string-based-keys). To collect aggregators of _different types_ with string-based labels, use [UntypedLabel](#untypedlabel-directory-of-different-types).

In strongly typed languages, the restriction to a single type allows nested objects to be extracted without casting.

### Indexing constructor and required members

```python
Index.ing(values)
```

  * `values` (list of present-tense aggregators) is the collection of aggregators to fill.
  * `entries` (mutable double) is the number of entries, initially 0.0.

### Indexed constructor and required members

```python
Index.ed(entries, values)
```

  * `entries` (double) is the number of entries.
  * `values` (list of past-tense aggregators) is the collection of filled aggregators.

### Fill and combine algorithms

```python
def fill(indexing, datum, weight):
    if weight > 0.0:
        for v in indexing.values:
            fill(v, datum, weight)
        indexing.entries += weight

def combine(one, two):
    if len(one.values) != len(two.values):
        raise Exception
    entries = one.entries + two.entries
    values = []
    for v1, v2 in zip(one.values, two.values):
        values.append(combine(v1, v2))
    return Index.ed(entries, values)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `type` (JSON string), name of the sub-aggregator type
  * `data` (JSON array of aggregators)

The fact that Index requires all contents to have a single type allows the `type` to be specified once here, instead of once for every item in the collection. [Branch](#branch-tuple-of-different-types) is more verbose.

**Example:**

```json
{"type": "Index",
 "data": {
   "entries": 123.0,
   "type": "Average",
   "data": [
     {"entries": 123.0, "mean": 3.14},
     {"entries": 123.0, "mean": 6.28},
     {"entries": 123.0, "mean": 99.9}]}}
```

## **Branch:** tuple of different types

Accumulate aggregators of different types, indexed by i0 through i9. Every sub-aggregator is filled with every input datum.

This primitive provides an anonymous collection of aggregators of _different types,_ usually for gluing together various statistics. For instance, if the following associates a sum of weights to every bin in a histogram,

```python
Bin.ing(100, 0, 1, lambda d: d.x,
  Sum.ing(lambda d: d.weight))
```

the following would associate the sum of weights and the sum of squared weights to every bin:

```python
Bin.ing(100, 0, 1, lambda d: d.x,
  Branch.ing(Sum.ing(lambda d: d.weight),
             Sum.ing(lambda d: d.weight**2)))
```

Branch is a basic building block for complex aggregators. The limitation to ten branches, indexed from i0 to i9, is a concession to type inference in statically typed languages. It is not a fundamental limit, but the type-metaprogramming becomes increasingly complex as branches are added. Error messages may be convoluted as the compiler presents internals of the type-metaprogramming in response to a user's simple mistake.

Therefore, individual implementations may allow more than ten branches, but the Histogrammar standard only requires ten.

To collect an unlimited number of aggregators of the _same type_ without naming them, use [Index](#index-list-with-integer-keys). To collect aggregators of the _same type_ with string-based labels, use [Label](#label-directory-with-string-based-keys). To collect aggregators of _different types_ with string-based labels, use [UntypedLabel](#untypedlabel-directory-of-different-types).

### Branching constructor and required members

```python
Branch.ing(values)
```

  * `values` (list of present-tense aggregators) is the collection of aggregators to fill.
  * `entries` (mutable double) is the number of entries, initially 0.0.

### Branched constructor and required members

```python
Branch.ed(entries, values)
```

  * `entries` (double) is the number of entries.
  * `values` (list of past-tense aggregators) is the collection of filled aggregators.

### Fill and combine algorithms

```python
def fill(branching, datum, weight):
    if weight > 0.0:
        for v in branching.values:
            fill(v, datum, weight)
        branching.entries += weight

def combine(one, two):
    if len(one.values) != len(two.values):
        raise Exception
    entries = one.entries + two.entries
    values = []
    for v1, v2 in zip(one.values, two.values):
        values.append(combine(v1, v2))
    return Branch.ed(entries, values)
```

### JSON fragment format

JSON object containing

  * `entries` (JSON number or "inf")
  * `data` (JSON array of JSON objects containing `type` (JSON string), name of sub-aggregator type and `data` (sub-aggregator))

**Example:**

```json
{"type": "Branch",
 "data": {
   "entries": 123.0,
   "data": [
     {"type": "Count", "data": 123.0},
     {"type": "Average", "data": {
       "entries": 123.0,
       "mean": 3.14}},
     {"type": "Bin", "data": {
       "low": -5.0,
       "high": 5.0,
       "entries": 123.0,
       "name": "position [cm]",
       "values:type": "Count",
       "values": [10.0, 20.0, 20.0, 30.0, 30.0],
       "underflow:type": "Count",
       "underflow": 5.0,
       "overflow:type": "Count",
       "overflow": 8.0,
       "nanflow:type": "Count",
       "nanflow": 0.0}}]}}
```

# Aliases: common functions and compositions

Although the following could be constructed by hand, they are so often used in data analyses that they have their own constructors. They are not distinct types, though: an aggregator created by explicit construction is completely interchangeable with an aggregator created by one of the following convenience functions.

## Identity

A transform (for [Count](#count-sum-of-weights)) named `"identity"` exists in the Histogrammar namespace with the following definition:

```python
def identity(weight):
    return weight
```

## Unweighted

A weighting function named `"unweighted"` exists in the Histogrammar namespace with the following definition:

```python
def unweighted(datum):
    return 1.0
```

## Histogram

An interval is divided into bins and the number of entries (sum of weights) is counted in each bin. All plotting front-ends should be capable of displaying this.

```python
def Histogram(num, low, high, quantity, selection=unweighted):
    return Select.ing(selection, Bin.ing(num, low, high, quantity,
        Count.ing(), Count.ing(), Count.ing(), Count.ing()))
```

  * `num` (32-bit integer) is the number of bins; must be at least one.
  * `low` (double) is the minimum-value edge of the first bin.
  * `high` (double) is the maximum-value edge of the last bin; must be strictly greater than `low`.
  * `quantity` (function returning double) computes the quantity to bin from the data.
  * `selection` (function returning boolean or double) computes the quantity to use as a selection (multiplicative factor on weight).

## SparselyHistogram

No memory is allocated for empty bins. This allows the analyst to plot a whole distribution without first knowing its support. (The analyst must have an appropriate choice of bin width, however.) Most plotting front-ends convert it into a dense representation immediately before plotting.

This kind of aggregator has two dangers: (1) if the `binWidth` is too small or the distribution has long tails, it can use a large amount of memory, and (2) if it has a few far outliers (e.g. 1e9 to express "missing value"), then the conversion to a dense representation can use a large amount of memory.

```python
def SparselyHistogram(binWidth, quantity, selection=unweighted, origin=0.0):
    return Select.ing(selection,
        SparselyBin.ing(binWidth, quantity, Count.ing(), Count.ing(), origin))
```

  * `binWidth` (double) is the width of a bin; must be strictly greater than zero.
  * `quantity` (function returning double) computes the quantity to bin from the data.
  * `selection` (function returning boolean or double) computes the quantity to use as a selection (multiplicative factor on weight).
  * `origin` (double) is the left edge of the bin whose index is 0.

## Profile

Views a two-dimensional dataset as a function by binning along one axis and averaging the other.

For a "profile plot" as defined in HBOOK, PAW, and ROOT, see [ProfileErr](#profileerr), which includes the variance in each bin to compute errors on the means.

```python
def Profile(num, low, high, binnedQuantity, averagedQuantity, selection=unweighted):
    return Select.ing(selection,
        Bin.ing(num, low, high, binnedQuantity,
            Average.ing(averagedQuantity)))
```

  * `num` (32-bit integer) is the number of bins; must be at least one.
  * `low` (double) is the minimum-value edge of the first bin.
  * `high` (double) is the maximum-value edge of the last bin; must be strictly greater than `low`.
  * `binnedQuantity` (function returning double) computes the quantity to bin from the data.
  * `averagedQuantity` (function returning double) computes the quantity to average from the data.
  * `selection` (function returning boolean or double) computes the quantity to use as a selection (multiplicative factor on weight).

## SparselyProfile

Views a two-dimensional dataset as a function by sparsely binning along one axis and averaging the other.

```python
def SparselyProfile(binWidth, binnedQuantity, averagedQuantity, selection=unweighted, origin=0.0):
    return Select.ing(selection,
        SparselyBin.ing(binWidth, binnedQuantity,
            Average.ing(averagedQuantity), Count.ing(), origin))
```

  * `binWidth` (double) is the width of a bin; must be strictly greater than zero.
  * `binnedQuantity` (function returning double) computes the quantity to bin from the data.
  * `averagedQuantity` (function returning double) computes the quantity to average from the data.
  * `selection` (function returning boolean or double) computes the quantity to use as a selection (multiplicative factor on weight).
  * `origin` (double) is the left edge of the bin whose index is 0.

## ProfileErr

Views a two-dimensional dataset as a function by binning along one axis and averaging the other, with variances to compute the error on the mean.

```python
def ProfileErr(num, low, high, binnedQuantity, averagedQuantity, selection=unweighted):
    return Select.ing(selection,
        Bin.ing(num, low, high, binnedQuantity,
            Deviate.ing(averagedQuantity)))
```

  * `num` (32-bit integer) is the number of bins; must be at least one.
  * `low` (double) is the minimum-value edge of the first bin.
  * `high` (double) is the maximum-value edge of the last bin; must be strictly greater than `low`.
  * `binnedQuantity` (function returning double) computes the quantity to bin from the data.
  * `averagedQuantity` (function returning double) computes the quantity to average from the data.
  * `selection` (function returning boolean or double) computes the quantity to use as a selection (multiplicative factor on weight).

## SparselyProfileErr

Views a two-dimensional dataset as a function by sparsely binning along one axis and averaging the other, with variances to compute the error on the mean.

```python
def SparselyProfileErr(binWidth, binnedQuantity, averagedQuantity, selection=unweighted, origin=0.0):
    return Select.ing(selection,
        SparselyBin.ing(binWidth, binnedQuantity,
            Deviate.ing(averagedQuantity), Count.ing(), origin))
```

  * `binWidth` (double) is the width of a bin; must be strictly greater than zero.
  * `binnedQuantity` (function returning double) computes the quantity to bin from the data.
  * `averagedQuantity` (function returning double) computes the quantity to average from the data.
  * `selection` (function returning boolean or double) computes the quantity to use as a selection (multiplicative factor on weight).
  * `origin` (double) is the left edge of the bin whose index is 0.

## TwoDimensionallyHistogram

Views a two-dimensional distribution in its entirety by counting the number of entries (sum of weights) in a grid of rectangular bins.

```python
def TwoDimensionallyHistogram(xnum, xlow, xhigh, xquantity,
                              ynum, ylow, yhigh, yquantity,
                              selection=unweighted):
    return Select.ing(selection,
        Bin.ing(xnum, xlow, xhigh, xquantity,
            Bin.ing(ynum, ylow, yhigh, yquantity)))
```

  * `xnum` (32-bit integer) is the number of bins along the x-axis; must be at least one.
  * `xlow` (double) is the minimum-value edge of the first x-axis bin.
  * `xhigh` (double) is the maximum-value edge of the last x-axis bin; must be strictly greater than `xlow`.
  * `xquantity` (function returning double) computes the quantity to bin along the x-axis from the data.
  * `ynum` (32-bit integer) is the number of bins along the y-axis; must be at least one.
  * `ylow` (double) is the minimum-value edge of the first y-axis bin.
  * `yhigh` (double) is the maximum-value edge of the last y-axis bin; must be strictly greater than `ylow`.
  * `yquantity` (function returning double) computes the quantity to bin along the y-axis from the data.
  * `selection` (function returning boolean or double) computes the quantity to use as a selection (multiplicative factor on weight).

## TwoDimensionallySparselyHistogram

Views a two-dimensional distribution in its entirety by counting the number of entries (sum of weights) in a conceptual grid of rectangular bins; the bins are only filled if non-zero.

```python
def TwoDimensionallySparselyHistogram(xbinWidth, xquantity,
                                      ybinWidth, yquantity,
                                      selection=unweighted,
                                      xorigin=0.0, yorigin=0.0):
    return Select.ing(selection,
        SparselyBin.ing(xbinWidth, xquantity,
            SparselyBin.ing(ybinWidth, yquantity,
                Count.ing(), Count.ing(), yorigin), Count.ing(), xorigin))
```

  * `xbinWidth` (double) is the width of a bin along the x-axis; must be strictly greater than zero.
  * `xquantity` (function returning double) computes the quantity to bin along the x-axis from the data.
  * `ybinWidth` (double) is the width of a bin along the y-axis; must be strictly greater than zero.
  * `yquantity` (function returning double) computes the quantity to bin along the y-axis from the data.
  * `selection` (function returning boolean or double) computes the quantity to use as a selection (multiplicative factor on weight).
  * `xorigin` (double) is the left edge of the bin on the x-axis whose index is 0.
  * `yorigin` (double) is the left edge of the bin on the y-axis whose index is 0.
