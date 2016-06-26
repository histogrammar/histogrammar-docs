---
title: Basic use in Python
type: default
toc: false
---

## Preliminaries

This tutorial demonstrates the use of the Python version of Histogrammar.  See the [installation guide](../../install) for installing version 0.7 or later.  

## Introduction

Histogrammar is an easy to use and powerful toolkit for aggregating data.  It
provides the building blocks and the back-end logic to make one or many
dimensional histograms, bar charts, box plots, efficiency plots, etc.   Before
aggregating any data, the user declaritively writes the structure of the
desired aggregation using Histogrammars simple and consistent grammar, and also
provides the rules to fill the structure with data as a user defined function
(usually a quick lambda function does the trick).  Histogrammar takes care of
the rest.  The user does not write the logic that actually makes the histogram,
no nested for loops, no map and reduce, no handling of bug-prone edge cases ---
this is taken care of by histogrammars back-end.

## First histogram

Histogrammars scope and usage is best demonstrated with a series of examples.
Lets make a histogram with 20 equally spaced bins of uniform random numbers,
whose range is from zero to one.  Let's call this histogrammars "hello world!":

```python
import histogrammar as hg

# generate a list of uniform random numbers
import random
data = [random.random() for i in range(100)]

# aggregation structure and fill rule
histogram = hg.Bin(num=20, low=0, high=1, quantity=lambda x: x, value=hg.Count())

# fill the histogram!
for d in data:
    histogram.fill(d)

# quick plotting convenvience method using matplotlib (if the user has this installed)
histogram.matplotlib()

import matplotlib.pyplot as plt
plt.title("hello world!")
```

### What happened here?

The first three arguements to `hg.Bin` are pretty self explanitory.  `num` is
the number of equal width bins to use.  `low` and `high` are the lower and
upper boundaries.

The last two arguements are more interesting.  `quantity` is the "fill rule": a
lambda or normal python function that is responsible for extracting the
quantity we are interested in aggregating from each data point in the data set.
Th




```python
events = EventIterator()
```
After loading the data set (`events`), you write:





























