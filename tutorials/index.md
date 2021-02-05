---
title: Tutorials
type: homepage
toc: false
---

## Basics

### Python

  * {% include progress.html complete=100 %} [Basic use in Python](https://nbviewer.jupyter.org/github/histogrammar/histogrammar-python/blob/master/histogrammar/notebooks/histogrammar_tutorial_basic.ipynb): Filling histograms from Numpy arrays and Pandas dataframes, composite histograms, plotting them with Matplotlib, many histograms at once, storing and retrieving them.
  * {% include progress.html complete=100 %} [Advanced use in Python](https://nbviewer.jupyter.org/github/histogrammar/histogrammar-python/blob/master/histogrammar/notebooks/histogrammar_tutorial_advanced.ipynb): Filling histograms from Spark dataframes, making and configuring many histograms at once from a dataframe.

### Scala

  * {% include progress.html complete=100 %} [Basic use in Scala](scala-basic): Histogrammar without any custom front-ends (for plotting) or back-ends (for aggregation). Focuses on the basics of making aggregations, what makes Histogrammar different, and ASCII-art plots.

<!-- ### Python -->

<!--   * {% include progress.html complete=0 %} [Basic use in Python](python-basic): Histogrammar without any custom front-ends (for plotting) or back-ends (for aggregation). Focuses on the basics of making aggregations, what makes Histogrammar different. -->

## Plotting front-ends

### Scala

  * {% include progress.html complete=100 %} [Making Bokeh plots in Spark](scala-spark-bokeh): How to aggregate Apache Spark data in Histogrammar and send it to the Bokeh plotting package in Scala.

### Python

  * {% include progress.html complete=50 %} [Making Matplotlib plots](python-matplotlib): How to send Histogrammar data to Matplotlib, the most popular Python plotting library.
  * {% include progress.html complete=100 %} [Making PyROOT plots](python-pyroot): How to send Histogrammar data to the ROOT analysis package in Python. This tutorial is complete enough that you could start here, if you are a ROOT user.
  * {% include progress.html complete=90 %} [Making Bokeh plots](python-bokeh): How to send Histogrammar data to the Bokeh plotting package in Python.

## Aggregation back-ends

### Scala

  * {% include progress.html complete=100 %} [Collecting data in Spark](scala-spark): How to use your Apache Spark cluster to make histograms, rather than downloading the data and plotting locally.
  * {% include progress.html complete=100 %} [Enhancements for SparkSQL](scala-sparksql): Special bindings to make histograms directly from Apache SparkSQL tables.
  <!-- * {% include progress.html complete=0 %} [Just-in-time compilation in Scala](scala-jit): How to make your aggregations on local data faster. -->

### Python

  * {% include progress.html complete=100 %} [Collecting data from Numpy](python-numpy): Aggregating over data in Numpy arrays without a Python for loop (i.e. faster).
  <!-- * {% include progress.html complete=0 %} [Collecting data from ROOT](python-rootjit): Aggregating over data in ROOT TTrees, taking advantage of JIT-compilation for 100X speed-ups. -->
  <!-- * {% include progress.html complete=0 %} [Collecting data from a GPU](python-gpu): Generating CUDA code to include in your GPU applications or filling data directly from Numpy arrays using PyCUDA. -->

## Utility applications

### Python

  <!-- * {% include progress.html complete=0 %} [HistogrammarAWK (hgawk)](python-hgawk): pipe data from grep, sed, and awk into Histogrammar to make plots on the UNIX shell. -->
  * {% include progress.html complete=90 %} [HistogrammarWatch (hgwatch)](python-hgwatch): stream aggregated data as JSON by appending to a file, a UNIX pipe, or through a socket or remote ssh connection to send interactive plots through any interface you can read as text.
