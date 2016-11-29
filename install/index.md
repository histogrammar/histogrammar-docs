---
title: Installation instructions
type: default
toc: true
summary: |
    <p>This page explains how to install Histogrammar in different ways. Use only the instructions relevant to your situation.</p>
scalaversion: 1.0.4
pythonversion: 1.0.6
---

# Get a specific release or the latest from GitHub

Starting in version 0.8, each language implementation of Histogrammar has a separate GitHub repository. Installation instructions (and project status) are given in each repository's README page. Reference documentation is generated from the source code for each release (and therefore isn't available for the latest).

[Read this](../specification/#version-numbers) for an explanation of version numbers and cross-compatibility.

| Specification | Scala | Python | C++ | Julia | R | Javascript |
|:-------------:|:-----:|:------:|:---:|:-----:|:-:|:----------:|
| [1.1-prerelease](../specification/1.1) | [repo](https://github.com/histogrammar/histogrammar-scala), [clone](https://github.com/histogrammar/histogrammar-scala.git), [zip](https://github.com/histogrammar/histogrammar-scala/archive/master.zip) ([README](http://github.com/histogrammar/histogrammar-scala/blob/master/README.md)) | [repo](https://github.com/histogrammar/histogrammar-python), [clone](https://github.com/histogrammar/histogrammar-python.git), [zip](https://github.com/histogrammar/histogrammar-python/archive/master.zip) ([README](http://github.com/histogrammar/histogrammar-python/blob/master/README.md)) | [repo](https://github.com/histogrammar/histogrammar-cpp), [clone](https://github.com/histogrammar/histogrammar-cpp.git), [zip](https://github.com/histogrammar/histogrammar-cpp/archive/master.zip) ([README](http://github.com/histogrammar/histogrammar-cpp/blob/master/README.md)) | [repo](https://github.com/histogrammar/Histogrammar.jl), [clone](https://github.com/histogrammar/Histogrammar.jl.git), [zip](https://github.com/histogrammar/Histogrammar.jl/archive/master.zip) ([README](http://github.com/histogrammar/Histogrammar.jl/blob/master/README.md)) | | |
| [1.0](../specification/1.0) | [{{ page.scalaversion }}](https://github.com/histogrammar/histogrammar-scala/releases/tag/{{ page.scalaversion }}) ([README](http://github.com/histogrammar/histogrammar-scala/blob/1.0.x/README.md), [reference](http://histogrammar.org/scala/{{ page.scalaversion }}/#org.dianahep.histogrammar.package)) | [{{ page.pythonversion }}](https://github.com/histogrammar/histogrammar-python/releases/tag/{{ page.pythonversion }}) ([README](http://github.com/histogrammar/histogrammar-python/blob/1.0.x/README.md), [reference](http://histogrammar.org/python/{{ page.pythonversion }}/)) | [1.0.0](https://github.com/histogrammar/histogrammar-cpp/releases/tag/1.0.0) ([README](http://github.com/histogrammar/histogrammar-cpp/blob/1.0.x/README.md)) | [1.0.0](https://github.com/histogrammar/Histogrammar.jl/releases/tag/1.0.0) ([README](http://github.com/histogrammar/Histogrammar.jl/blob/1.0.x/README.md)) | | |
| [0.8](../specification/0.8) | [0.8.0](https://github.com/histogrammar/histogrammar-scala/releases/tag/0.8.0) ([README](http://github.com/histogrammar/histogrammar-scala/blob/0.8.x/README.md), [reference](http://histogrammar.org/scala/0.8.0/#org.dianahep.histogrammar.package)) | [0.8.0](https://github.com/histogrammar/histogrammar-python/releases/tag/0.8.0) ([README](http://github.com/histogrammar/histogrammar-python/blob/0.8.x/README.md), [reference](http://histogrammar.org/python/0.8.0/)) | [0.8.0](https://github.com/histogrammar/histogrammar-cpp/releases/tag/0.8.0) ([README](http://github.com/histogrammar/histogrammar-cpp/blob/0.8.x/README.md)) | [0.8.0](https://github.com/histogrammar/Histogrammar.jl/releases/tag/0.8.0) ([README](http://github.com/histogrammar/Histogrammar.jl/blob/0.8.x/README.md)) | | |
| [0.7](../specification/0.7) | [0.7.1](https://github.com/histogrammar/histogrammar-multilang/releases/tag/0.7.1) ([reference](http://histogrammar.org/scala/0.7.1/#org.dianahep.histogrammar.package)) | [0.7.1](https://github.com/histogrammar/histogrammar-multilang/releases/tag/0.7.1) ([reference](http://histogrammar.org/python/0.7.1/)) | [0.7.1](https://github.com/histogrammar/histogrammar-multilang/releases/tag/0.7.1) | | | |
| 0.6 | [0.6](https://github.com/histogrammar/histogrammar-multilang/releases/tag/0.6) ([reference](http://histogrammar.org/scala/0.6/#org.dianahep.histogrammar.package)) | [0.6](https://github.com/histogrammar/histogrammar-multilang/releases/tag/0.6) | [0.6](https://github.com/histogrammar/histogrammar-multilang/releases/tag/0.6) | | | |

# Install from a public repository

## Java/Scala or Apache Spark

<a href="http://search.maven.org/#search|ga|1|histogrammar">Histogrammar is available on Maven Central</a>, a publicly accessible Java/Scala repository with dependency management.

### Apache Spark

To use Histogrammar in the Spark shell, you don't have to download anything. Just start Spark with

```bash
spark-shell --packages "org.diana-hep:histogrammar_2.11:{{ page.scalaversion }}"
```

and call

```scala
import org.dianahep.histogrammar._
```

on the Spark prompt. For plotting with Bokeh, include `org.diana-hep:histogrammar-bokeh_2.11:{{ page.scalaversion }}` and for interaction with Spark-SQL, include `org.diana-hep:histogrammar-sparksql_2.11:{{ page.scalaversion }}`.

Use `_2.11` for compatibility with Spark 2.x (Scala 2.11) and `_2.10` for compatibility with Spark 1.x (Scala 2.10).

**Note:** due to a [dependency bug](https://github.com/bokeh/bokeh-scala/issues/28), Bokeh is incompatible with Spark 2.x (Scala 2.11).

### Java/Scala with Maven

To compile Histogrammar into a project with the Maven build tool, add

```xml
<dependency>
  <groupId>org.diana-hep</groupId>
  <artifactId>histogrammar_2.11</artifactId>
  <version>{{ page.scalaversion }}</version>
</dependency>
```

to your `<dependencies>` section. Use `_2.11` for compatibility with Scala 2.11 and `_2.10` for compatibility with Scala 2.10.

### Scala with sbt

To use Histogrammar in `sbt console` or to compile it into a project with the sbt build tool, add

```scala
libraryDependencies += "org.diana-hep" %% "histogrammar" % "{{ page.scalaversion }}"
```

to your `build.sbt` file. The double-percent gets the appropriate version of Histogrammar for your version of Scala.

### Quick start

In fact, the easiest way to start an interactive Scala session with histogrammar is simply to make the following `build.sbt`:

```scala
page.scalaversion := "2.11.8"
libraryDependencies += "org.diana-hep" %% "histogrammar" % "{{ page.scalaversion }}"
```

and run `sbt console`. You don't need to install Scala or anything other than [sbt](http://www.scala-sbt.org/download.html).

## Python

<a href="https://pypi.python.org/pypi/Histogrammar/">Histogrammar is available on PyPI</a>, a publicly accessible Python repository with dependency management.

### If you have superuser (root) access

To install the latest version of Histogrammar, use

```bash
sudo easy_install histogrammar
```

or

```bash
sudo pip histogrammar
```

depending on whether you have `pip` installed (recommended). Some systems with both Python 2 and 3 use `easy_install3` and `pip3` to distinguish the Python 3 version.

On freshly minted Ubuntu machines, you can install `pip` with

```bash
sudo apt-get install python-setuptools
easy_install pip
```

### If you do not have superuser access

```bash
pip install --user histogrammar
```

which installs it in `~/.local` (Python knows where to find it).

### For use in PySpark

PySpark uses both Histogrammar-Python (as an interface) and Histogrammar-Scala (for faster calculations). To use it, you need to download Histogrammar-Python as described immediately above, and launch PySpark with a request for Histogrammar-Scala:

```bash
pyspark --packages "org.diana-hep:histogrammar_2.11:{{ page.scalaversion }}"
```

Use `_2.11` for compatibility with Spark 2.x (Scala 2.11) and `_2.10` for compatibility with Spark 1.x (Scala 2.10).

In PySpark, you should be able to call

```python
from histogrammar import *
import histogrammar.sparksql
histogrammar.sparksql.addMethods(df)
```

where `df` is a `DataFrame` that you would like to enable with Histogrammar. You can now call

```python
h = df.Bin(100, -5.0, 5.0, df["plotme"] + df["andme"])
```

to get a histogram `h` of Column expression `df["plotme"] + df["andme"]`. All of the processing is performed in Java with Spark's DataFrame optimizations.

## C++

When available, [Spack](https://github.com/LLNL/spack) instructions will be found here.

## Julia

When available, [Julia package](http://pkg.julialang.org/) instructions will be found here.

## R

When available, [CRAN](https://cran.r-project.org/) instructions will be found here.

## Javascript

When available, [npm](https://www.npmjs.com/) instructions will be found here.
