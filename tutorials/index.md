---
title: Tutorials
type: homepage
toc: false
---

<div style="float: right; width: 160px; font-size: 12px; border: solid 1px #4e5f70;"><p style="margin: 5px;">Tutorials with no progress bar are <i>complete.</i></p></div>

## Basics

  * [Installation](../install): Always the first step. This page is subdivided by project, so you can install as little or as much as you need (e.g. Python but no Scala, etc.).

### Scala

  * [Basic use in Scala](scala-basic): Histogrammar without any custom front-ends (for plotting) or back-ends (for aggregation). Focuses on the basics of making aggregations, what makes Histogrammar different, and ASCII-art plots.

### Python

  * {% include progress.html complete=0 %} [Basic use in Python](python-basic): Histogrammar with Matplotlib, the most popular Python plotting library, and no custom back-ends (for aggregation). Focuses on the basics of making aggregations, what makes Histogrammar different.

## Plotting front-ends

### Scala

  * [Making Bokeh plots in Spark](scala-spark-bokeh): How to aggregate Apache Spark data in Histogrammar and send it to the Bokeh plotting package in Scala.

### Python

  * [Making PyROOT plots](python-pyroot): How to send Histogrammar data to the ROOT analysis package in Python. This tutorial is complete enough that you could start here.
  * {% include progress.html complete=0 %} [Making Bokeh plots](python-bokeh): How to send Histogrammar data to the Bokeh plotting package in Python.

## Aggregation back-ends

### Scala

  * {% include progress.html complete=0 %} [Collecting data in Spark](scala-spark): How to use your Apache Spark cluster to make histograms, rather than downloading the data and plotting locally.
  * {% include progress.html complete=0 %} [Enhancements for SparkSQL](scala-sparksql): Special bindings to make histograms directly from Apache SparkSQL tables.
  * {% include progress.html complete=0 %} [Just-in-time compilation in Scala](scala-jit): How to make your aggregations on local data faster.

### Python

  * {% include progress.html complete=0 %} [Enhancements for Numpy](python-numpy): Aggregating over data in Numpy arrays without a Python for loop (i.e. faster).

## Utility applications

### Python

  * {% include progress.html complete=0 %} [HistogrammarAWK (hgawk)](python-hgawk): pipe data from grep, sed, and awk into Histogrammar to make plots on the UNIX shell.
  * {% include progress.html complete=90 %} [HistogrammarWatch (hgwatch)](python-hgwatch): stream aggregated data as JSON by appending to a file, a UNIX pipe, or through a socket or remote ssh connection to send interactive plots through any interface you can read as text.

## Subroutines

These tutorials contain instructions common to most of the others. If you need one, you'll be referred to it, but they're listed here for completeness.

  * [CMS public dataset in Scala](scala-cmsdata): Sample data for plots in Scala.
  * [CMS public dataset in Python](python-cmsdata): The same sample data, accessed in Python.

