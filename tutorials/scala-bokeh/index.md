---
title: Drawing plots in Scala with Bokeh
type: default
toc: false
summary: |
    <p>If you're working with Histogrammar in Scala and want to use <a href="https://github.com/bokeh/bokeh-scala">Bokeh</a> to draw plots, read this page.</p>
---

## Setting up

The examples on this page have been tested with Histogrammar 0.7. Any subsequent version should work. See the [Installation instructions](../install) if you need to install it.

It also uses the [CMS public dataset](scala-cmsdata) as sample data.

## Plotting a Histogram

Following is an example of plotting a simple histogram with `scala-bokeh` in the interactive spark-shell (Spark context and SQL context are available as `sc` and `sqlContext`). Following assumes that `Bokeh` and `histogrammar` jars are included in the classpath:	

```scala
import org.dianahep.histogrammar._
import org.dianahep.histogrammar.bokeh._
```

Explore the file `data/triggerIsoMu24_50fb-1.json` used for the test:

```bash
{"photons": [], "MET": {"px": -20.33991813659668, "py": -31.165260314941406}, "electrons": [], "jets": [], "muons": [{"E": 46.11058044433594, "pz": 38.082550048828125, "px": 21.024599075317383, "py": 15.292481422424316, "q": 1, "iso": 0.0}], "numPrimaryVertices": 3}
```

In this example, let us produce a plot of muon momentum. Thus, the only variables we are going to need are:

```bash
"muons": [{"pz": 38.082550048828125, "px": 21.024599075317383, "py": 15.292481422424316}]
```

Define JSON schema to read only that subset of rows on demand (more advanced approaches use `Avro` or `Parquet`):
```scala
val jsonSchema = sqlContext.read.json(sc.parallelize(Array("""{"muons": [{"pz": 38.082550048828125, "px": 21.024599075317383, "py": 15.292481422424316}]}""")))
```
 
Next, define case classes which would allow casting these data to the desired type:

```scala
case class Mu(px: Double, py: Double, pz: Double)
case class MuWrapper(muons: Array[Mu])
```

Now map the input data to your case classes and load it into Spark's Dataset.  Columns are automatically lined up by name, and the types are preserved:

```scala
import sqlContext.implicits._
val dataset = sqlContext.read.format("json").schema(jsonSchema.schema).load("file:///.../histogrammar-docs/data/triggerIsoMu24_50fb-1.json").as[MuWrapper].cache()
```

In this example, data is first flattened using a `flatMap` transformation and then a set of selection requirements is applied using a `filter` action with a predicate:
```scala
val data_rdd = dataset.flatMap(muon => muon.muons).filter(muon => (muon.pz > 2.0)).rdd
```

After data extraction and transformation is completed, the histogram is booked and filled:

```scala
val p_histogram = Histogram(100, 0, 200, {mu: Mu => math.sqrt(mu.px*mu.px + mu.py*mu.py + mu.pz*mu.pz)})
val final_histogram = data_rdd.aggregate(p_histogram)(new Increment, new Combine)
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

val data_rdd1 = dataset.flatMap(muon => muon.muons).filter(muon => (muon.pz > 2.0)).rdd
val data_rdd2 = dataset.flatMap(muon => muon.muons).filter(muon => (muon.pz > 20.0)).rdd

val p_histogram1 = Histogram(100, 0, 200, {mu: Mu => math.sqrt(mu.px*mu.px + mu.py*mu.py + mu.pz*mu.pz)})
val p_histogram2 = Histogram(100, 0, 200, {mu: Mu => math.sqrt(mu.px*mu.px + mu.py*mu.py + mu.pz*mu.pz)})

val selection1= data_rdd1.aggregate(p_histogram1)(new Increment, new Combine)
val selection2= data_rdd2.aggregate(p_histogram2)(new Increment, new Combine)

val G1 = selection1.bokeh()
val G2 = selection2.bokeh(glyphType="circle",glyphSize=3,fillColor=Color.Blue)

plot(G1,G2)
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
val G1 = selection1.bokeh()
val G2 = selection2.bokeh(glyphType="circle",glyphSize=3,fillColor=Color.Blue)
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
import org.dianahep.histogrammar._
import org.dianahep.histogrammar.bokeh._

val jetSchema = sqlContext.read.json(sc.parallelize(Array("""{"jets":[{"pz": 42.1006965637207, "px": -13.06346321105957, "py": 32.3252067565918}]}""")))

val muonSchema = sqlContext.read.json(sc.parallelize(Array("""{"muons": [{"pz": 38.082550048828125, "px": 21.024599075317383, "py": 15.292481422424316}]}""")))

case class Mu(px: Double, py: Double, pz: Double)
case class MuWrapper(muons: Array[Mu])
case class Jet(px: Double, py: Double, pz: Double)
case class JetWrapper(jets: Array[Jet])

import sqlContext.implicits._
val dataset1 = sqlContext.read.format("json").schema(muonSchema.schema).load("file:///.../histogrammar-docs/data/triggerIsoMu24_50fb-1.json").as[MuWrapper].cache()

val dataset2 = sqlContext.read.format("json").schema(jetSchema.schema).load("file:///.../histogrammar-docs/data/triggerIsoMu24_50fb-1.json").as[JetWrapper].cache()

import io.continuum.bokeh._
val data_rdd1 = dataset1.flatMap(muon => muon.muons).filter(muon => (muon.pz > 2.0)).rdd
val data_rdd2 = dataset2.flatMap(jet => jet.jets).filter(jet => (jet.pz > 3.0)).rdd
```

When booking the histograms, make sure the binning of the histograms to be stacked is the same, otherwise an exception will be thrown:

```scala
val p_histogram1 = Histogram(100, 0, 200, {mu: Mu => math.sqrt(mu.px*mu.px + mu.py*mu.py + mu.pz*mu.pz)})
val p_histogram2 = Histogram(100, 0, 200, {jet: Jet => math.sqrt(jet.px*jet.px + jet.py*jet.py + jet.pz*jet.pz)})

val sample1 = data_rdd1.aggregate(p_histogram1)(new Increment, new Combine)
val sample2 = data_rdd2.aggregate(p_histogram2)(new Increment, new Combine)

val s = Stack.build(sample1,sample2)
val mystackplot = s.bokeh().plot()
save(mystackplot,"stackplot.html")
```
