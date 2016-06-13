---
title: Basic use in Scala
type: default
toc: false
---

## Preliminaries

This tutorial uses the Scala version of Histogrammar. See the [installation guide](../../install) for installing version 0.7 or later.

It also uses the [CMS public dataset](../scala-cmsdata). Set up an iterator named `events` in your console. You will need a network connection.

## First histogram

```scala
import org.dianahep.histogrammar._

val pt = Bin(10, 0, 100, {muon: Muon => muon.pt})

val events = EventIterator()
events.take(1000) flatMap {event: Event => event.muons} foreach {muon: Muon => pt.fill(muon)}

import org.dianahep.histogrammar.ascii._

pt.println
```

