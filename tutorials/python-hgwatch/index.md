---
title: Histogrammar Watch (hgwatch)
type: default
toc: false
summary: |
    <p>If you're performing an analysis on a remote computer and want to display plots locally, or if you're working in one framework (like Scala) and want to make plots with another (like Python), read this page.</p>
---

## Preliminaries

This tutorial uses the Python version and optionally the Scala version of Histogrammar. See the [installation guide](../../install) for installing version 0.7 or later. If you install the Python version in a non-standard directory, be sure that your `PATH` environment variable is properly set.

It uses artificial datasets because the focus is on data transfer, not data.

## Overview

When you install the Python version of Histogrammar with `sudo python setup.py install`, it adds or overwrites an executable named `hgwatch`, or Histogrammar Watch.

This executable watches a file, a UNIX pipe, a socket, or an ssh connection for lines of text containing the JSON of aggregated histograms. Whenever a new line appears, it runs a chosen set of commands that usually plot the histogram. This way, you can keep a local window open to work that you're doing remotely&mdash; either on a remote computer or in a different framework from the plotting framework.

Histogrammar Watch doubles as a Python prompt, so that you can investigate your plot objects if anything looks wrong.

### Help text

As a first step, verify that you have `hgwatch` and view its help text by typing,

```bash
hgwatch --help
```

```
usage: hgwatch [-h] [-f FILE] [-c COMMAND] [-s ADDRESS:PORT] [-p COMMANDS]
               [--ssh USER@HOST:PORT] [--ssh-key LOCAL-FILE] [--json] [--root]

Watch a file, pipe, or remote connection for Histogrammar JSON objects and
perform some action on each (such as plotting).

optional arguments:
  -h, --help            show this help message and exit
  -f FILE               File or pipe to watch (default is "-", standard
                        input).
  -c COMMAND            Shell command(s) to run and watch; if a filename, use
                        the contents of that file.
  -s ADDRESS:PORT       Socket address and port to watch (separated by a
                        colon).
  -p COMMANDS           Python commands to run on each new histogram "hg"; if
                        a filename, use the contents of that file (default is
                        "print(hg)").
  --ssh USER@HOST:PORT  Connect to remote host for -f and -c (no effect for -s
                        or "-f -") with the paramiko library for ssh. Raises
                        an exception if paramiko is not installed.
  --ssh-key LOCAL-FILE  Identity for ssh (RSA or DSA private key file) or
                        None.
  --json                append -p with print-out of JSON.
  --root                append -p with visualization in ROOT.

Only one of [-f, -c, -s] may be used. Each JSON object in the stream must be a
separate line of text, and no action is performed until the input buffer
flushes with an end of line character ("\n").
```

### Simple example

Open two terminals in the same directory and type the following into one of them:

```bash
cat > intermediate.json
```

This starts writing to a file named `intermediate.json`, appending to the file every time you hit the return key.

In the other terminal, type

```bash
hgwatch -c 'tail -f intermediate.json' -p 'print(hg)'
```

This opens a Python prompt with a background thread running `tail -f intermediate.json`. Go back to the first terminal and type

```json
{"type": "Count", "data": 123}
```

and press enter. This JSON represents one of the simplest aggregators possible: a count with a value of 123. In the other terminal, you should see

```
<Count 123.0>
```

every time you add one to the first terminal. If you don't, then you probably have a problem with buffers not flushing: don't worry about it until you try some of the other examples below (depending on the type of your operating system, one method may work while another doesn't).

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
hgwatch -f intermediate.json -p 'print(hg)'
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

Histogrammar Watch applies a given set of commands to each new plot, which was `print(hg)` in our examples above. But it also gives you a Python prompt to do whatever you want. Hit the enter key a few times to get the traditional Python `>>>` prompt to show up (since the background thread may have obscured it) and do

```python
hg.entries
```

or

```python
hg + hg
```

to pull it apart. You have full control over this `hg` object, so you can inspect it if anything goes wrong. For instance, it might have a form that your plotting commands don't correctly handle; this can help you find out why and correct your plotting commands.

If the processes that is emitting plots are not emitting the right format, you can look at this raw input with `hgin` (a string).

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


TODO


## Remote viewing

Now let's handle a data transfer over the network. Histogrammar Watch has a `-s` option for monitoring a socket (IP address and port), but let's consider the more common case where you're not a system administrator and are not allowed to open ports.

You can watch a file over ssh using the `--ssh` (and maybe `--ssh-key`) option.

TODO
