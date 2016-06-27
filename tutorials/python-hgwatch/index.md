---
title: HistogrammarWatch (hgwatch)
type: default
toc: false
summary: |
    <p>If you're performing an analysis on a remote computer and want to display plots locally, or if you're working in one framework (like Scala) and want to make plots with another (like Python), read this page.</p>
    <p><b>Author:</b> <a href="http://github.com/jpivarski">Jim Pivarski</a></p>
---

## Preliminaries

This tutorial uses the Python version and optionally the Scala version of Histogrammar. See the [installation guide](../../install) for installing version 0.7 or later. If you install the Python version in a non-standard directory, be sure that your `PATH` environment variable is properly set.

It uses artificial datasets because the focus is on data transfer, not data.

## Overview

When you install the Python version of Histogrammar with `sudo python setup.py install`, it adds or overwrites an executable named `hgwatch`, also known as HistogrammarWatch.

This executable watches a file, a UNIX pipe, a socket, or an ssh connection for lines of text containing the JSON of aggregated histograms. Whenever a new line appears, it runs a chosen set of commands that plot the histogram (or do something user-defined). This way, you can keep a local window open to work that you're doing remotely&mdash; either on a remote computer or in a different framework from the plotting framework.

HistogrammarWatch can also double as a Python prompt, so that you can investigate your plot objects if anything looks wrong.

### Help text

As a first step, verify that you have `hgwatch` and view its help text by typing,

```bash
hgwatch --help
```

You should see

```
usage: hgwatch [-h] [-f FILE] [-c COMMAND] [-s ADDRESS:PORT] [-p COMMANDS]
               [-i] [--json] [--root] [-v]

Watch a file, pipe, or remote connection for Histogrammar JSON objects and
perform some action on each (such as plotting).

optional arguments:
  -h, --help       show this help message and exit
  -f FILE          File or pipe to watch (default is "-", standard input).
  -c COMMAND       Shell command(s) to run and watch; if a filename, use the
                   contents of that file.
  -s ADDRESS:PORT  Socket address and port to watch (separated by a colon).
  -p COMMANDS      Python commands to run on each new histogram "hg"; if a
                   filename, use the contents of that file.
  -i               Start an interactive Python prompt with the watcher in a
                   background thread.
  --json           append -p with print-out of JSON.
  --root           append -p with visualization in ROOT.
  -v, --version    show program's version number and exit

Only one of [-f, -c, -s] may be used. Each JSON object in the stream must be a
separate line of text, and no action is performed until the input buffer
flushes with an end of line character ("\n").
```

(Version 0.7; later versions may have more options.)

### Simple example

Open two terminals in the same directory and type the following into one of them:

```bash
cat > intermediate.json
```

This starts writing to a file named `intermediate.json`, appending to the file every time you hit the return key.

In the other terminal, type

```bash
hgwatch -i -c 'tail -f intermediate.json' -p 'print(hg)'
```

This opens a Python prompt (`-i`) with a background thread running `tail -f intermediate.json`. Go back to the first terminal and type

```json
{"type": "Count", "data": 123}
```

and press enter. This JSON represents one of the simplest aggregators possible: a count with a value of 123. In the other terminal, you should see

```
<Count 123.0>
```

every time you add a line to the first terminal. If you don't (e.g. because you're using Python 3), then you probably have a problem with buffers not flushing: don't worry about it until you try some of the other examples below (depending on your system, one method may work while another doesn't).

What's happening here is that `hgwatch` is running `print(hg)` for each `hg` object that it sees in `intermediate.json`. If instead of a simple count, we had a complex histogram, and instead of simply printing it, we plotted it, we would now have a pipeline from a process that can only produce text to another that can show you graphical visualizations of your data, on demand.

### Named pipes

The shared file has a few deficiencies. For one thing, if reading or writing is buffered, you wouldn't see the plots right away, which is a show-stopper for an interactive viewer. For another, it saves everything you've ever looked at to disk, using more disk space than you might want to use. And finally, disk access is slower than memory&mdash; why add that bottleneck to the workflow?

UNIX and its descendants have an infrequently used feature called "named pipes." You can create one in your directory with

```bash
mkfifo intermediate.json
```

You can write to the named pipe with one process and read from it with another without touching disk. It is equivalent to (but more convenient than) piping one process's standard out into the other's standard in.

Let's try the above example with a named pipe. Create it as above and write to it in the first terminal:

```bash
cat > intermediate.json
```

Then in the other terminal, run

```bash
hgwatch -i -f intermediate.json -p 'print(hg)'
```

Both the `cat` and `hgwatch -f` (f for file) are pretending that `intermediate.json` is a file, but if you made it with `mkfifo`, it's a pipe. Now when you add

```json
{"type": "Count", "data": 123}
```

to it, you should see

```
<Count 123.0>
```

in the other without touching disk. Python 3 is more likely to work in this mode.

### Interacting with the data

HistogrammarWatch applies a given set of commands to each new plot, which was `print(hg)` in our examples above. But if you pass the `-i` flag, it also gives you a Python prompt to do whatever you want. Hit the enter key a few times to get the traditional Python `>>>` prompt to show up (since the background thread may have obscured it) and do

```python
hg.entries
```

or

```python
hg + hg
```

to pull it apart or manipulate it. You have full control over this `hg` object, so you can inspect it if anything goes wrong. For instance, it might have a form that your plotting commands don't correctly handle; this can help you find out why and correct your plotting commands.

If the processes that is emitting plots are not emitting the right format, you can look at this raw input with `hgin` (a string). For instance,

```python
import json
json.loads(hgin)
```

can help you find JSON formatting errors. Additionally, there's an `hgerr` for the last Exception (if any), and you can use

```python
import traceback
traceback.print_last()
```

to see the full stack trace.

## Cross-platform plots

Now let's handle a more complex case: producing plots in Scala and viewing them with Python's PyROOT.

In the above examples, we opened the named pipe for writing (`cat > intermediate.json`) before reading (`hgwatch -i -f intermediate.json`). If you do it the other way around, `hgwatch` "hangs" until the writer starts. Let's do that.

```bash
python scripts/hgwatch -i -f intermediate.json --root
```

The `--root` option sets up a set of commands that plot the data in ROOT. If you do not have ROOT on your system, this will fail. See `--help` for a list of available plotting front-ends: you can substitute another for this exercise.

In the other terminal, start a Scala prompt with the Histogrammar JAR loaded (at least version 0.7):

```bash
scala -cp target/histogrammar-0.7.jar
```

Import the Histogrammar classes and make a histogram.

```scala
import org.dianahep.histogrammar._

val b = Bin(100, -3, 3, {x: Unit => scala.util.Random.nextGaussian})
```

This one takes no input (Scala `()`, type `Unit`, which is `void` in Java) and fills with a random value. Now open a connection to the named pipe and dump the histogram into it.

```scala
val out = new JsonDump("intermediate.json")
out.append(b)
```

That should cause HistogrammarWatch to pop up a window showing an empty histogram. Back on the Scala side, add some data:

```scala
b.fill(())
b.fill(())
b.fill(())
out.append(b)
```

You should see this in the window.

```scala
for (i <- 0 until 1000)
  b.fill(())
out.append(b)

// animation!
for (i <- 0 until 1000) {
  b.fill(())
  out.append(b)
}
```

It is as though you had an interactive ROOT window in Scala, rather than Python.

## Remote plotting through ssh

Now let's handle a data transfer over the network. HistogrammarWatch has a `-s` option for monitoring a socket (IP address and port), which is useful if the remote computer has no firewall or you're allowed to open ports. Often, though, all you have is ssh, so let's consider that case first.

Open two terminals, one on your local computer and the other on the remote site where you want to analyze data. The local computer will display plots with Python's PyROOT and the remote site will generate them in Scala. (Again, you can substitute the front-end and back-end if you find another more convenient; this tutorial only concerns the data flow.)

On the remote computer, create a named pipe and start a Scala process with the Histogrammar JAR.

```bash
mkfifo intermediate.json
scala -cp target/histogrammar-0.7.jar
```

On the local computer, start a HistogrammarWatch with ssh as its command:

```bash
hgwatch -i -c "ssh REMOTE -x cat intermediate.json --root"
```

where `REMOTE` is the address of the remote computer (and supply a user name, etc., if you need to). If your ssh prompts you with a password, you can't use the `-i` (interactive) option, since the Python input loop interferes with the password prompt.

Now make a histogram and stream its output through `intermediate.json`, as before:

```scala
import org.dianahep.histogrammar._

val b = Bin(100, -3, 3, {x: Unit => scala.util.Random.nextGaussian})

val out = new JsonDump("intermediate.json")
out.append(b)

b.fill(())
b.fill(())
b.fill(())
out.append(b)

for (i <- 0 until 1000)
  b.fill(())
out.append(b)

// animation!
for (i <- 0 until 1000) {
  b.fill(())
  out.append(b)
}
```

You should see a stream of histogram plots.

### Multi-hop ssh

That's fine for a single ssh connection, but some working environments are so locked down that you have to log into one machine just to log into another, where the actual analysis is performed. Almost the same solution can be used for that; you just need to set up your `~/.ssh/config`.

If you don't already have a `~/.ssh/config` file, create an empty one. Then add

```config
Host StepA
    Hostname StepA

Host StepB
    ProxyCommand ssh StepA -W %h:%p

Host StepC
    ProxyCommand ssh StepB -W %h:%p
```

for a connection from your local computer to `StepA`, from there to `StepB`, and from there to `StepC`, etc. Any number of steps can be chained this way.

Now when you

```bash
hgwatch -i -c "ssh REMOTE -x cat intermediate.json --root"
```

use the final destination (`StepC` in the above example) as your `REMOTE`. If any of the steps require a password, you can't use the `-i` (interactive) option.

If you're familiar with ssh configuration files, you can also add user names, custom ports, identity files (for passwordless ssh), etc.

## Remote plotting through a socket

**FIXME: not implemented.**
