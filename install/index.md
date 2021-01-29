---
title: Installation instructions
type: default
toc: true
summary: |
    <p>This page explains how to install Histogrammar in different ways. Use only the instructions relevant to your situation.</p>
scalaversion: 1.0.11
pythonversion: 1.0.12
---

# Install from a public repository

## Python

<a href="https://pypi.python.org/pypi/Histogrammar/">Histogrammar is available on PyPI</a>, a publicly accessible Python repository with dependency management.

### If you have superuser (root) access

To install the latest version of Histogrammar, use

```bash
sudo easy_install histogrammar
```

or

```bash
sudo pip install histogrammar
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
pyspark --packages "io.github.histogrammar:histogrammar_2.12:1.0.11"
```

Use `_2.11` for compatibility with Spark 2.x (Scala 2.11).

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


## Java/Scala or Apache Spark

<a href="http://search.maven.org/#search|ga|1|histogrammar">Histogrammar is available on Maven Central</a>, a publicly accessible Java/Scala repository with dependency management.

### Apache Spark

To use Histogrammar in the Spark shell, you don't have to download anything. Just start Spark with

```bash
spark-shell --packages "io.github.histogrammar:histogrammar_2.12:1.0.11"
```

and call

```scala
import org.dianahep.histogrammar._
```

on the Spark prompt. For interaction with Spark-SQL, include `io.github.histogrammar:histogrammar-sparksql_2.12:1.0.11`.

Use `_2.11` for compatibility with Spark 2.x (Scala 2.11).


### Java/Scala with Maven

To compile Histogrammar into a project with the Maven build tool, add

```xml
<dependency>
  <groupId>io.github.histogrammar</groupId>
  <artifactId>histogrammar_2.12</artifactId>
  <version>1.0.11</version>
</dependency>
```

to your `<dependencies>` section. Use `_2.11` for compatibility with Scala 2.11.

### Scala with sbt

To use Histogrammar in `sbt console` or to compile it into a project with the sbt build tool, add

```scala
libraryDependencies += "io.github.histogrammar" %% "histogrammar" % "1.0.11"
```

to your `build.sbt` file. The double-percent gets the appropriate version of Histogrammar for your version of Scala.

### Quick start

In fact, the easiest way to start an interactive Scala session with histogrammar is simply to make the following `build.sbt`:

```scala
page.scalaversion := "2.12.13"
libraryDependencies += "io.github.histogrammar" %% "histogrammar" % "1.0.11"
```

and run `sbt console`. You don't need to install Scala or anything other than [sbt](http://www.scala-sbt.org/download.html).
