---
title: Tutorials
type: homepage
toc: false
---

## Utilities

Other pages refer back to these for set-up that would otherwise have to be repeated.

  * {% include progress.html complete=100 %} [Installation](../install): The first step.
  * {% include progress.html complete=100 %} [CMS public dataset in Scala](scala-cmsdata): Sample data for plots.
  * {% include progress.html complete=100 %} [CMS public dataset in Python](python-cmsdata): The same sample data, accessed in Python.

## Basic tutorials

### Scala

  * {% include progress.html complete=0 %} [Basic use in Scala](scala-basic): Histogrammar without any custom frontends (for plotting) or backends (for aggregation). Focuses on the basics of making aggregations, what makes Histogrammar different, and ASCII-art plots.

### Python

  * {% include progress.html complete=0 %} [Basic use in Python](python-basic): Histogrammar without any custom frontends (for plotting) or backends (for aggregation). Focuses on the basics of making aggregations, what makes Histogrammar different, and ASCII-art plots.
  * {% include progress.html complete=70 %} [Plotting on the commandline](python-commandline): How to pipe data from grep, sed, and awk into Histogrammar in UNIX shell scripts.

## Plotting frontends

### Scala

  * {% include progress.html complete=30 %} [Making Bokeh plots](scala-bokeh): How to send Histogrammar data to the [Bokeh plotting package in Scala](http://github.com/bokeh/bokeh-scala).

### Python

  * {% include progress.html complete=0 %} [Making Bokeh plots](python-bokeh): How to send Histogrammar data to the [Bokeh plotting package in Python](http://bokeh.pydata.org/en/latest/).
  * {% include progress.html complete=0 %} [Making ROOT plots](python-root): How to send Histogrammar data to the [ROOT analysis package in Python](http://root.cern.ch/).
  * {% include progress.html complete=0 %} [Making Matplotlib plots](python-root): How to send Histogrammar data to the [Matplotlib plotting package](http://matplotlib.org/).

## Aggregation backends

### Scala

  * {% include progress.html complete=0 %} [Collecting data in Spark](scala-spark): How to use your [Apache Spark](http://spark.apache.org/) cluster to make histograms, rather than downloading the data and plotting locally.
  * {% include progress.html complete=0 %} [Enhancements for SparkSQL](scala-sparksql): Special bindings to make histograms directly from [Apache SparkSQL](http://spark.apache.org/sql/) tables.
  * {% include progress.html complete=0 %} [Just-in-time compilation in Scala](scala-jit): How to make your aggregations on local data faster.

### Python

  * {% include progress.html complete=0 %} [Enhancements for Numpy](python-numpy): Aggregating over data in Numpy arrays without a Python for loop (i.e. faster).

