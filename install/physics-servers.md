---
title: Pre-installed versions on physics servers
type: default
toc: true
summary: |
    <p>This page points to versions of Histogrammar installed at Fermilab's LPC and CERN's LXPLUS servers. Use this page if you have access to one of these systems.</p>
---

## LPC at Fermilab

If you have access to the LPC at Fermilab, you can run a preinstalled copy of Histogrammar version 0.7.1.

| LPC at Fermilab | `cmslpc-sl6.fnal.gov` |
|:----------------|:----------------------|
| for Scala | `/uscms/home/pivarski/public/histogrammar-0.7.1.jar` |
| for SparkSQL | `/uscms/home/pivarski/public/histogrammar-sparksql-0.7.1.jar          ` |
| for Python 2.6 | `/uscms/home/pivarski/public/histogrammar0.7-python2.6` |
| for C++ | `/uscms/home/pivarski/public/include07` |
| Scala 2.10.5 | `/uscms/home/pivarski/public/scala-2.10.5` |
| Spark 1.6.1 | `/uscms/home/pivarski/public/spark-1.6.1-bin-hadoop1` |

To start a Scala prompt with Histogrammar loaded, do the following:

```bash
export PATH=/uscms/home/pivarski/public/scala-2.10.5/bin:$PATH
scala -cp /uscms/home/pivarski/public/histogrammar-0.7.1.jar
```
```scala
scala> import org.dianahep.histogrammar._
```

To start a local (non-distributed, testing) Spark session with Histogrammar loaded, do the following:

```bash
export PATH=/uscms/home/pivarski/public/spark-1.6.1-bin-hadoop1/bin:$PATH
spark-shell --jars=/uscms/home/pivarski/public/histogrammar-0.7.1.jar,/uscms/home/pivarski/public/histogrammar-sparksql-0.7.1.jar
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

```cpp
#include "histogrammar.hpp"
```

in your code.

## LXPLUS at CERN

If you have access to LXPLUS at CERN, you can run a preinstalled copy of Histogrammar version 0.7.1.

| LXPLUS at CERN | `lxplus.cern.ch` |
|:----------------|:----------------------|
| for Scala | `/afs/cern.ch/user/p/pivarski/public/histogrammar-0.7.1.jar` |
| for SparkSQL | `/afs/cern.ch/user/p/pivarski/public/histogrammar-sparksql-0.7.1.jar` |
| for Python 2.6 | `/afs/cern.ch/user/p/pivarski/public/histogrammar0.7-python2.6` |
| for C++ | `/afs/cern.ch/user/p/pivarski/public/include07` |
| Scala 2.10.5 | `/afs/cern.ch/user/p/pivarski/public/scala-2.10.5` |
| Spark 1.6.1 | `/afs/cern.ch/user/p/pivarski/public/spark-1.6.1-bin-hadoop1` |

To start a Scala prompt with Histogrammar loaded, do the following:

```bash
export PATH=/afs/cern.ch/user/p/pivarski/public/scala-2.10.5/bin:$PATH
scala -cp /afs/cern.ch/user/p/pivarski/public/histogrammar-0.7.1.jar
```
```scala
scala> import org.dianahep.histogrammar._
```

To start a local (non-distributed, testing) Spark session with Histogrammar loaded, do the following:

```bash
export PATH=/afs/cern.ch/user/p/pivarski/public/spark-1.6.1-bin-hadoop1/bin:$PATH
spark-shell --jars=/afs/cern.ch/user/p/pivarski/public/histogrammar-0.7.1.jar,/afs/cern.ch/user/p/pivarski/public/histogrammar-sparksql-0.7.1.jar
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

```c
#include "histogrammar.hpp"
```
in your code.
