---
title: Installation instructions
type: default
toc: true
summary: |
    <p>This page explains how to install Histogrammar or Histogrammar components in different ways. Use only the instructions relevant to your situation.</p>
---

# Quick-install from a public repository

## Histogrammar-Python from PyPI (pip)

**FIXME:** when Histogrammar reaches v1.0, it will be ready to upload to PyPI.

## Histogrammar-Scala from Maven Central (sbt, Maven, Spark)

**FIXME:** when Histogrammar reaches v1.0, it will be ready to upload to Maven Central.

## Histogrammar-R from CRAN

**FIXME:** Histogrammar-R doesn't exist yet. Change that.

## Histogrammar-Javascript from npm

**FIXME:** Histogrammar-Javascript doesn't exist yet. Change that.

## Histogrammar-C++ from Spack

**FIXME:** when Histogrammar reaches v1.0, it will be ready to upload to [Spack](https://github.com/LLNL/spack) (C++ packaging system).

# Releases maintained on physics servers

## Quick start for LPC at Fermilab

If you have access to the LPC at Fermilab, you can run a preinstalled copy of [Histogrammar version 0.7](http://github.com/diana-hep/histogrammar/releases/tag/0.7).

| LPC at Fermilab | `cmslpc-sl6.fnal.gov` |
|:----------------|:----------------------|
| for Scala | `/uscms/home/pivarski/public/histogrammar-0.7.jar` |
| for SparkSQL | `/uscms/home/pivarski/public/histogrammar-sparksql-0.7.jar          ` |
| for Python 2.6 | `/uscms/home/pivarski/public/histogrammar0.7-python2.6` |
| for C++ | `/uscms/home/pivarski/public/include07` |
| Scala 2.10.5 | `/uscms/home/pivarski/public/scala-2.10.5` |
| Spark 1.6.1 | `/uscms/home/pivarski/public/spark-1.6.1-bin-hadoop1` |

To start a Scala prompt with Histogrammar loaded, do the following:

```bash
export PATH=/uscms/home/pivarski/public/scala-2.10.5/bin:$PATH
scala -cp /uscms/home/pivarski/public/histogrammar-0.7.jar
```
```scala
scala> import org.dianahep.histogrammar._
```

To start a local (non-distributed, testing) Spark session with Histogrammar loaded, do the following:

```bash
export PATH=/uscms/home/pivarski/public/spark-1.6.1-bin-hadoop1/bin:$PATH
spark-shell --jars=/uscms/home/pivarski/public/histogrammar-0.7.jar,/uscms/home/pivarski/public/histogrammar-sparksql-0.7.jar
```
```scala
scala> import org.dianahep.histogrammar._
scala> import org.dianahep.histogrammar.sparksql._
```

To use Histogrammar in Python, do:

```bash
export PYTHONPATH=/uscms/home/pivarski/public/histogrammar0.7-python2.6:$PYTHONPATH
python
```
```python
>>> from histogrammar import *
```

To use Histogrammar in a C++ project, add:

```
-I /uscms/home/pivarski/public/include07
```

to your compiler options and

```c++
#include "histogrammar.hpp"
```

in your code.

## Quick start for LXPLUS at CERN

If you have access to LXPLUS at CERN, you can run a preinstalled copy of [Histogrammar version 0.7](http://github.com/diana-hep/histogrammar/releases/tag/0.7).

| LXPLUS at CERN | `lxplus.cern.ch` |
|:----------------|:----------------------|
| for Scala | `/afs/cern.ch/user/p/pivarski/public/histogrammar-0.7.jar` |
| for SparkSQL | `/afs/cern.ch/user/p/pivarski/public/histogrammar-sparksql-0.7.jar` |
| for Python 2.6 | `/afs/cern.ch/user/p/pivarski/public/histogrammar0.7-python2.6` |
| for C++ | `/afs/cern.ch/user/p/pivarski/public/include07` |
| Scala 2.10.5 | `/afs/cern.ch/user/p/pivarski/public/scala-2.10.5` |
| Spark 1.6.1 | `/afs/cern.ch/user/p/pivarski/public/spark-1.6.1-bin-hadoop1` |

To start a Scala prompt with Histogrammar loaded, do the following:

```bash
export PATH=/afs/cern.ch/user/p/pivarski/public/scala-2.10.5/bin:$PATH
scala -cp /afs/cern.ch/user/p/pivarski/public/histogrammar-0.7.jar
```
```scala
scala> import org.dianahep.histogrammar._
```

To start a local (non-distributed, testing) Spark session with Histogrammar loaded, do the following:

```bash
export PATH=/afs/cern.ch/user/p/pivarski/public/spark-1.6.1-bin-hadoop1/bin:$PATH
spark-shell --jars=/afs/cern.ch/user/p/pivarski/public/histogrammar-0.7.jar,/afs/cern.ch/user/p/pivarski/public/histogrammar-sparksql-0.7.jar
```
```scala
scala> import org.dianahep.histogrammar._
scala> import org.dianahep.histogrammar.sparksql._
```

To use Histogrammar in Python, do:

```bash
export PYTHONPATH=/afs/cern.ch/user/p/pivarski/public/histogrammar0.7-python2.6:$PYTHONPATH
python
```
```python
>>> from histogrammar import *
```

To use Histogrammar in a C++ project, add:
```
-I/afs/cern.ch/user/p/pivarski/public/include07
```
to your compiler options and

```c++
#include "histogrammar.hpp"
```
in your code.

# Download and compile from GitHub

## Download source code from a fixed release

See Histogrammar's [GitHub release page](http://github.com/diana-hep/histogrammar/releases). Below the status message for each release (which includes a table of implemented primitives) is a zip and a tar.gz.

Fixed releases are stable but may not have all the features you're looking for. Each tutorial declares an earliest working version; if that version doesn't exist, it means that you need to download or clone the latest source code.

## Download or clone source code from the bleeding edge

Here is a [zip file for downloading](https://github.com/diana-hep/histogrammar/archive/master.zip).

To clone the release, use

```sh
git clone https://github.com/diana-hep/histogrammar.git
```

If you are a GitHub user, you can also fork [the repository](http://github.com/diana-hep/histogrammar) to propose pull requests. If you're testing a new feature, you may have been directed to [a particular branch](http://github.com/diana-hep/histogrammar/branches/active).

## Install Histogrammar-Python from source

### System-wide

Use the standard [Python](https://www.python.org/downloads/) install script:

```sh
cd histogrammar/python
sudo python setup.py install
```

Histogrammar is compatible with both Python 2.7 and Python 3.4+. There is experimental support for Python 2.6 in the latest source.

### In a user directory

As usual with Python setup scripts, you can install software in your own directory without superuser access,

```sh
python setup.py install --home=~/my_installation_directory
```

but then you have to make sure that `PYTHONPATH` includes `my_installation_directory/lib`.

## Compile Histogrammar-Scala and all dependent projects

Run [Maven](https://maven.apache.org/download.cgi) from the base directory. The parent `pom.xml` will compile the sub-projects in the right order.

```sh
cd histogrammar
mvn install
```

As usual with Maven, the compiled JAR files go into a `target` directory for each project (with dependencies in `target/lib`) and are cached in your `~/.m2/repository`.

## Compile Histogrammar-Scala subprojects manually

If you need to compile and install them separately, here is the dependency order:

  * `scala` is the base package; you must `cd` to this directory and `mvn install` here first.
    * `scala-bokeh` depends on `scala`; you can `cd` to this directory and `mvn install` here after the first installation succeeded.
    * `scala-jit` depends on `scala`; you can `cd` to this directory and `mvn install` here after the first installation succeeded.
    * `scala-sparksql` depends on `scala`; you can `cd` to this directory and `mvn install` here after the first installation succeeded.

Thus, if one of the dependent projects doesn't compile on your system (`scala-jit` requires a C compiler and make tools) and you don't need it, you can compile the others manually.

Subprojects find the base project's JAR in your `~/.m2/repository`.
