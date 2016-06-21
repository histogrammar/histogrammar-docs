---
title: Basic use in Python
type: default
toc: false
---

## Preliminaries

This tutorial demonstrates the use of the Python version of Histogrammar.  See the [installation_guide](../../install) for installing version 0.7 or later.

```python
events = EventIterator()
```

## First histogram

The prototypical use case for histogrammar is, of course, to summarize data as a histogram.  Using the `Bin` object, we define the number of bins (10), their range (0 to 100), and how the data that falls within a bin to be aggregated (Count()).  

After loading the data set (`events`), you write:

```python
import histogrammar as hg

histogram = hg.Bin(10, 0, 100, lambda event: event.met.pt)
# or equivalently
histogram = hg.Bin(10, 0, 100, lambda event: event.met.pt, hg.Count())

for event in events:
    histogram.fill(event)
```






























