---
title: Drawing plots in Python with Bokeh
type: default
toc: false
summary: |
    <p>If you're working with in Python and want to use <a href="https://github.com/bokeh/bokeh">Bokeh</a> to draw plots, read this page.</p>
    <p><b>Author:</b> <a href="https://github.com/ASvyatkovskiy">Alexey Svyatkovskiy</a></p>
---

## Setting up

The examples on this page have been tested with Histogrammar 0.7. Any subsequent version should work. See the [Installation instructions](../install) if you need to install it.

## Plotting a Histogram in `Python`

The examples of plotting histograms with `python-bokeh` presented in this section use `Python` and artificial data.
In the Python (or PySpark REPL), start by importing the Histogrammar package and the plotting library:

```python
from histogrammar import *
from histogrammar.plot.bokeh import plot,save,view
```

Generate artificial data:

```python
simple = [3.4, 2.2, -1.8, 0.0, 7.3, -4.7, 1.6, 0.0, -3.0, -1.7]
```

Book two histograms:

```python
one = Histogram(5, -5.0, 8.0, lambda x: x)
two = Histogram(5, -3.0, 7.0, lambda x: x)
```

Fill both histograms in one line of code using `Label` class:

```python
labeling = Label(one=one, two=two)
for _ in simple: labeling.fill(_)
```

Start by plotting histogram `one`:

```python
glyph_one = one.bokeh()
plot_one = plot(glyph_one)
save(plot_one,"python_plot_one.html")
```

### Configuring Bokeh Glyph attributes

By default, a line glyph of black color is plotted. One can easily turn this into a bar plot filled with red color by passing arguments to `bokeh()` method as follows:

```python
glyph_one = one.bokeh(glyphType="histogram",fillColor="red")
```

In addition, default axes titles ('x' and 'y') can be changed to, for instance:
```python
plot_one = plot("x [cm]","Num. events",glyph_one)
save(plot_one,"python_plot_one.html")
```

### Superimposing multiple glyphs on one plot

To superimpose two histograms booked and filled above on one plot, one create and configure a glyph for each of the histograms, and call the `plot()` method which accepts variable length argument list, and therefore can take any number of glyphs.

```python
glyph_one = one.bokeh() #use defaults
glyph_two = two.bokeh(glyphType="histogram",fillColor="red") #customize
plot_both = plot(glyph_one,glyph_two)
save(plot_both,"python_plot_both.html")
```

### Adding a legend

Having a `GlyphRenderer` (a type of object returned by the `bokeh()` method) and a `Plot` (a type of object returned by the `plot()` method) objects one can easily put a `Legend` onto the plot using built-in `Bokeh` tools. For instance, given the histograms from the previous example:

```python
glyph_one = one.bokeh()
glyph_two = two.bokeh(glyphType="histogram",fillColor="red")

from bokeh.models import Legend
legend = Legend(legends=[("curve1", [glyph_one]), ("curve2", [glyph_two])])
plot_both = plot(glyph_one,glyph_two)
plot_both.add_layout(legend)
save(plot_both,"python_plot_both_legend.html")
```


### Plotting a stack of Histograms

Here is an example of how to make a stacked plot of histograms. Let us generate more artificial data, different from one and two:

```python
extra = [3.2, 3.2, -2.1, 1.0, 1.3, -3.4, 0.6, 0.0, -1.0, 1.7]
```
and book a third histogram:

```python
three = Histogram(5, -3.0, 7.0, lambda x: x)
```
Note: only histograms with the same binning can be stacked!

Now, fill it:
```python
map(lambda _: three.fill(_),extra)
```

Prepare a stacked histogram using a dedicated `build()` method, and plot it: 
```python
s = Stack.build(two,three)
glyph_stack = s.bokeh() #use defaults
plot_stack = plot(glyph_stack)
save(plot_stack,"python_plot_stack.html")
```

## Bokeh server

The architecture of `Bokeh` plotting library is such that the objects representing plots, ranges, axes, and glyphs are created in `Python` (or `Scala`), and then converted to a JSON format that is consumed by the client library, `BokehJS`. 


### `Bokeh` server capabilities

Bokeh server makes it possible for model objects described above in `Python` and in the browser to be in sync with one another, allowing for additional features: 

* respond to UI and tool events generated in a browser
* push updates of the widgets or plots in a browser

Following example shows how to run plotting applications with Bokeh server locally:

```python
from histogrammar import *
from histogrammar.plot.bokeh import plot,save,view

simple = [3.4, 2.2, -1.8, 0.0, 7.3, -4.7, 1.6, 0.0, -3.0, -1.7]

one = Histogram(5, -5.0, 8.0, lambda x: x)
two = Histogram(5, -3.0, 7.0, lambda x: x)

labeling = Label(one=one, two=two)
for _ in simple: labeling.fill(_)

glyph_one = one.bokeh()
plot_one = plot(glyph_one)
view(plot_one)
```

This application can be shared it with other people, so they can run it locally themselves in the same manner. 

### Standalone Bokeh server

One might also want to deploy the Bokeh server plotting applications in a way that other people can access it. This section describes some of the considerations that arise in that case. It is possible to simply run the Bokeh server on a network for users to interact with directly at a specified port (5006 by default).

### Nginx server

If the goal is to serve an web application to the general Internet, it is often desirable to host the application on an internal network, and proxy connections to it through some dedicated HTTP server. 
Using Nginx HTTP server with reverse-proxying is one of the ways to achieve that.

Following is a configuration file
that sets up an Nginx server to proxy incoming connections to 127.0.0.1 on port 80 to 127.0.0.1:5100 internally. 

```bash
server {
    listen 80 default_server;
    server_name _;

    access_log  /tmp/bokeh.access.log;
    error_log   /tmp/bokeh.error.log debug;

    location / {
        proxy_pass http://127.0.0.1:5100;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host:$server_port;
        proxy_buffering off;
    }

}
```
To work in this configuration, one would need to use some of the command line options to configure the Bokeh Server. In particular, to use `--port` to specify that the Bokeh Server should listen itself on port 5100 as well as the `--host` option to whitelist 127.0.0.1:80 as an acceptable host on the incoming request header:

```bash
serve myapp.py --port 5100 --host 127.0.0.1:80
```
