---
title: Drawing plots in Scala with Bokeh
type: default
toc: false
summary: |
    <p>If you're working with <a href="http://spark.apache.org/">Apache Spark</a> in Scala and want to use <a href="https://github.com/bokeh/bokeh-scala">Bokeh</a> to draw plots, read this page.</p>
---

## Setting up

The examples on this page have been tested with Histogrammar 0.7. Any subsequent version should work. See the [Installation instructions](../install) if you need to install it.

It also uses Apache Spark, so you'll want to start a Spark Shell with the following JARs loaded:

  * histogrammar-0.7.jar
  * histogrammar-bokeh-0.7.jar
  * Bokeh and all of its dependencies.

If you have compiled from source and are in the `histogrammar/scala-bokeh` directory, a convenient way to get a `--jars` argument for Spark is with the following:

```bash
spark-shell --jars=`ls target/**/*.jar | tr '\n' ','`
```

(lists all JARs, concatenates them with commas, and passes to Spark). Use whatever other options are necessary for your Spark installation, such as a custom `--master`.

This tutorial also uses the [CMS public dataset](scala-cmsdata) as sample data. Load the code on that page to get an `events` iterator, then do:

```scala
val dataset_rdd = sc.parallelize(events)
```

to turn it into a Spark RDD. It may take about 20 seconds to transfer all the data to your Spark cluster.

## Plotting a Histogram

Following is an example of plotting a simple histogram with `scala-bokeh` in the interactive spark-shell (Spark context and SQL context are available as `sc` and `sqlContext`). Following assumes that `Bokeh` and `histogrammar` jars are included in the classpath:	

```scala
import org.dianahep.histogrammar._
import org.dianahep.histogrammar.bokeh._
```

In this example, we plot muon quantities, so extract the muons to their own RDD:

```scala
val muons_rdd = dataset_rdd.flatMap(_.muons).filter(_.pz > 2.0)
```

After data extraction and transformation is completed, the histogram is booked and filled:

```scala
val p_histogram = Histogram(100, 0, 200, {mu: Muon => math.sqrt(mu.px*mu.px + mu.py*mu.py + mu.pz*mu.pz)})
val final_histogram = muons_rdd.aggregate(p_histogram)(new Increment, new Combine)
```

Users are strongly encouraged to learn the syntax of `Bokeh` package, especially about `Glyph` and `Plot` abstractions. Plotting a one dimensional histogram can be done in two simple lines of code:

```scala
val myfirstplot = final_histogram.bokeh().plot()
save(myfirstplot,"myfirstplot.html")
```

The resulting plot is saved to an HTML file and can be viewed and interactively edited in a browser.

#### Configuring Bokeh Glyph attributes

The above example uses default parameters and styles for the histograms plotted. A number of the attributes can be configured, including glyph type (line or a marker), marker style (e.g. circle, diamond shape), sizes and colors of glyphs. 
Import Bokeh libraries to be able to configure glyph colors:

```scala
import io.continuum.bokeh._

val mysecondplot = final_histogram.bokeh(glyphType="circle",glyphSize=3,fillColor=Color.Blue).plot()
save(mysecondplot,"mysecondplot.html")
```

### Example: superimposing multiple histograms on one plot

To superimpose two or more histograms on a single `Bokeh` plot one can simply create and customize glyphs
for each of the histograms, and then use `plot()` method passing all of the glyphs as arguments like:

```scala
import io.continuum.bokeh._

val p_histogram1 = muons_rdd.aggregate(Histogram(100, 0, 200, {mu: Muon => math.sqrt(mu.px*mu.px + mu.py*mu.py + mu.pz*mu.pz)}, {mu: Muon => mu.pz > 2.0}))(new Increment, new Combine)

val p_histogram2 = muons_rdd.aggregate(Histogram(100, 0, 200, {mu: Muon => math.sqrt(mu.px*mu.px + mu.py*mu.py + mu.pz*mu.pz)}, {mu: Muon => mu.pz > 20.0}))(new Increment, new Combine)

val G1 = p_histogram1.bokeh()
val G2 = p_histogram2.bokeh(glyphType="circle",glyphSize=3,fillColor=Color.Blue)

val mythirdplot = plot(G1,G2)
save(mythirdplot,"mythirdplot.html")
```

Here, `plot()` method accepts variable length argument list, and therefore can take any number of glyphs. An alternative API:
```scala
def plot(xLabel:String, yLabel: String, glyphs: GlyphRenderer*)
```
also allows to configure axes titles.

### Specifying a legend

Having a `GlyphRenderer` (a type of object returned by the `bokeh()` method) and a `Plot` (a type of object returned by the `plot()` method) objects one can easily put a `Legend` onto the plot using built-in `Bokeh` tools. For instance, given the histograms from the previous example:

```
val G1 = p_histogram1.bokeh()
val G2 = p_histogram2.bokeh(glyphType="circle",glyphSize=3,fillColor=Color.Blue)
val legend = List("curve1" -> List(G1),"curve2" -> List(G2))

val plots = plot(G1,G2)
val leg = new Legend().plot(plots).legends(legend)
plots.renderers <<= (leg :: _)
save(plots,"mythirdplot_legend.html")
```

### Example: plotting a SparselyHistogram

Same API can be used to plot sparsely binned histograms. 

## Stack

### Example: plotting a stack of histograms

Here is an example of how to make a stacked plot of two histograms. The most common use case in particle physics is to plot various simulated samples for the same final state. Here, a somewhat artificial case is considered when muon and jet momenta from the same sample are considered:

```scala
val jets_rdd = dataset_rdd.flatMap(_.jets).filter(_.pz > 3.0)
```

When booking the histograms, make sure the binning of the histograms to be stacked is the same, otherwise an exception will be thrown:

```scala
val p_histogram1 = muons_rdd.aggregate(Histogram(100, 0, 200, {mu: Muon => math.sqrt(mu.px*mu.px + mu.py*mu.py + mu.pz*mu.pz)}))(new Increment, new Combine)

val p_histogram2 = jets_rdd.aggregate(Histogram(100, 0, 200, {jet: Jet => math.sqrt(jet.px*jet.px + jet.py*jet.py + jet.pz*jet.pz)}))(new Increment, new Combine)

val s = Stack.build(p_histogram1,p_histogram2)
val mystackplot = plot(s.bokeh(): _*)
save(mystackplot,"stackplot.html")
```
