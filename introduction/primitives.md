## Catalog of primitives

**Zeroth kind:** primitives that ignore data and depend only on weights.

  * [Count](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Count$): Count data, ignoring their content. (Actually a sum of weights.)

**First kind:** primitives that aggregate a given scalar quantity, without sub-aggregators.

  * [Sum](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Sum$): Accumulate the sum of a given quantity.
  * [Average](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Average$): Accumulate the weighted mean of a given quantity.
  * [Deviate](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Deviate$): Accumulate a weighted variance, mean, and total weight of a given quantity (using an algorithm that is stable for large numbers).
  * [Minimize](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Minimize$): Find the minimum value of a given quantity. If no data are observed, the result is NaN.
  * [Maximize](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Maximize$): Find the maximum value of a given quantity. If no data are observed, the result is NaN.

**Second kind:** primitives that group by a given scalar quantity, passing data to a sub-aggregator.

  * [Bin](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Bin$): Split a given quantity into equally spaced bins between specified limits and fill only one bin per datum.
  * [SparselyBin](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.SparselyBin$): Split a quantity into equally spaced bins, filling only one bin per datum and creating new bins as necessary.
  * [CentrallyBin](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.CentrallyBin$): Split a quantity into bins defined by a set of bin centers, filling only one datum per bin with no overflows or underflows.
  * [IrregularlyBin](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.AdaptivelyBin$): Split a quanity into bins dynamically with a clustering algorithm, filling only one datum per bin with no overflows or underflows.
  * [Categorize](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Categorize$): Split a given quantity by its categorical (string-based) value and fill only one category per datum.
  * [Fraction](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Fraction$): Accumulate two containers, one with all data (denominator), and one with data that pass a given selection (numerator).
  * [Stack](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Stack$): Accumulate a suite containers, filling all that are above a given cut on a given expression.

**Third kind:** primitives that act as containers, passing data to all sub-aggregators.

  * [Select](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Select$): Accumulate an aggregator for data that satisfy a cut (or more generally, a weighting).
  * [Limit](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Limit$): Accumulate an aggregator until its number of entries reaches a predefined limit.
  * [Label](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Label$): Accumulate any number of containers of the SAME type and label them with strings. Every one is filled with every input datum.
  * [UntypedLabel](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.UntypedLabel$): Accumulate containers of any type except Count and label them with strings. Every one is filled with every input datum.
  * [Index](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Index$): Accumulate any number of containers of the SAME type anonymously in a list. Every one is filled with every input datum.
  * [Branch](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Branch$): Accumulate containers of DIFFERENT types, indexed by i0 through i9. Every one is filled with every input datum.

**Fourth kind:** primitives that collect sets of raw data.

  * [Bag](https://histogrammar.github.io/histogrammar-docs/scala/latest/index.html#org.dianahep.histogrammar.Bag$): Accumulate raw numbers, vectors of numbers, or strings, merging identical values.
