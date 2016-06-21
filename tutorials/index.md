---
title: Tutorials
type: homepage
toc: false
---

These first few contain instructions common to most of the other tutorials below.

  * {% include progress.html complete=100 %} [Installation](../install): The first step.
  * {% include progress.html complete=100 %} [CMS public dataset in Scala](scala-cmsdata): Sample data for plots.
  * {% include progress.html complete=100 %} [CMS public dataset in Python](python-cmsdata): The same sample data, accessed in Python.

## Basics

### Scala

  * {% include progress.html complete=100 %} [Basic use in Scala](scala-basic): Histogrammar without any custom front-ends (for plotting) or back-ends (for aggregation). Focuses on the basics of making aggregations, what makes Histogrammar different, and ASCII-art plots.

### Python

  * {% include progress.html complete=0 %} [Basic use in Python](python-basic): Histogrammar with [Matplotlib](http://matplotlib.org/), the most popular Python plotting library, and no custom back-ends (for aggregation). Focuses on the basics of making aggregations, what makes Histogrammar different.

## Plotting front-ends

### Scala

  * {% include progress.html complete=30 %} [Making Bokeh plots](scala-bokeh): How to send Histogrammar data to the [Bokeh plotting package in Scala](http://github.com/bokeh/bokeh-scala).

### Python

  * {% include progress.html complete=0 %} [Making Bokeh plots](python-bokeh): How to send Histogrammar data to the [Bokeh plotting package in Python](http://bokeh.pydata.org/en/latest/).
  * {% include progress.html complete=100 %} [Making PyROOT plots](python-pyroot): How to send Histogrammar data to the [ROOT analysis package in Python](http://root.cern.ch/).

## Aggregation back-ends

### Scala

  * {% include progress.html complete=0 %} [Collecting data in Spark](scala-spark): How to use your [Apache Spark](http://spark.apache.org/) cluster to make histograms, rather than downloading the data and plotting locally.
  * {% include progress.html complete=0 %} [Enhancements for SparkSQL](scala-sparksql): Special bindings to make histograms directly from [Apache SparkSQL](http://spark.apache.org/sql/) tables.
  * {% include progress.html complete=0 %} [Just-in-time compilation in Scala](scala-jit): How to make your aggregations on local data faster.

### Python

  * {% include progress.html complete=0 %} [Enhancements for Numpy](python-numpy): Aggregating over data in Numpy arrays without a Python for loop (i.e. faster).

## Utilities

### Python

  * {% include progress.html complete=0 %} [HistogrammarAWK (hgawk)](python-hgawk): pipe data from grep, sed, and awk into Histogrammar to make plots on the UNIX shell.
  * {% include progress.html complete=90 %} [HistogrammarWatch (hgwatch)](python-hgwatch): stream aggregated data as JSON by appending to a file, a UNIX pipe, or through a socket or remote ssh connection to send interactive plots through any interface you can read as text.
