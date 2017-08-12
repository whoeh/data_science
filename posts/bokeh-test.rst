.. title: Bokeh Test
.. slug: bokeh-test
.. date: 2017-05-24 12:29:09 UTC-07:00
.. tags: bokeh plotting
.. category: plotting
.. link: 
.. description: A test of using bokeh in a post.
.. type: text

The Plot
--------
   
.. raw:: html

   <script
       src="portland_unemployment.js"
       id="686c5dd6-168a-4f7d-acbc-524875d93b59"
       data-bokeh-model-id="c473232a-dc2c-4b75-988c-f9bc6517b4b9"
       data-bokeh-doc-id="402d8e3c-1595-4d65-9f76-e11068c629ab"
   ></script>         

What This Is
------------

This is a re-do of the final plot done for data-science with python course 2 week 4. The original was done with matplotlib and this was done with bokeh to get some interaction working. When I tried to create it the first time bokeh raised some errors saying that ``height`` had been defined more than once. I don't know what really caused it - possibly a namespace clash where I was re-using something I didn't intend to re-use - but when I created a new notebook that only created the one plot it worked. Since this uses javascript I used Jupyter and the web-inteface to test it out (emacs ipython doesn't seem to be able to render javascript (unless I'm doing it wrong)).

How It Got Exported
-------------------

I won't go over the creating of the data (since I just copied it from an earlier notebook) but this is how the bokeh plot was created.

Imports
~~~~~~~

These were the bokeh parts I needed.

.. code:: python

   from bokeh.models import (
       BoxAnnotation,
       CustomJS,
       Span,
       Toggle,
   )

   from bokeh.io import (
       output_file,
       output_notebook,
       show,
   )

   from bokeh.plotting import (
       figure,
       ColumnDataSource,
    )

   from bokeh.models import (
       CrosshairTool,
       HoverTool,
       PanTool,
       ResetTool,
       ResizeTool,
       SaveTool,
       UndoTool,
       WheelZoomTool,
       )

   from bokeh.layouts import column       
   from bokeh.resources import CDN
   from bokeh.embed import autoload_static

Some Constants
~~~~~~~~~~~~~~

.. code:: python

   NATIONAL_COLOR = "slategrey"
   NATIONAL_LABEL = "National"
   PORTLAND_COLOR = "cornflowerblue"
   PORTLAND_LABEL = "Portland-Hillsboro-Vancouver"
   S_AND_P_COLOR = "#90151B"
   S_AND_P_LABEL = "S & P 500 Index"
   HOUSING_COLOR = "#D89159"
   HOUSING_LABEL = "House Price Index"


The Data
~~~~~~~~

``bokeh`` doesn't work with ``pandas`` ``DataFrame``'s (or at least I couldn't get it to work). Instead you create a DataFrame-like object using the ``ColumnDataSource``.


.. code:: python

   portland_source = ColumnDataSource(
       data=dict(
           month_data=portland.datetime,
           unemployment=portland.unemployment_rate,
           month_label=portland.date,
           )
   )
   
   national_source = ColumnDataSource(
       data=dict(
           month_data=national.datetime,
           unemployment=national.unemployment_rate,
           month_label=national.date,
           )
       )
   
   housing_source = ColumnDataSource(
       data=dict(
           month_data=house_price_index.datetime,
           price=house_price_index.price,
           month_label=s_and_p_index.date,
           )
       )
       
   s_and_p_source = ColumnDataSource(
       data=dict(
           month_data=s_and_p_index.datetime,
           value=s_and_p_index.VALUE,
           month_label=s_and_p_index.date,
           )
   )    


The Tools
~~~~~~~~~

These are the things that add interactivity to the plot. You have to create new ones for each figure so I made a function to get them.


.. code:: python

   def make_tools():
       """makes the tools for the figures
       
       Returns:
        list: tool objects
       """
       hover = HoverTool(tooltips=[
       ("month", "@month_label"),
       ("unemployment", "@unemployment"),
       ])
       
       tools = [
           hover,
           CrosshairTool(),
           PanTool(),
           ResetTool(),
           ResizeTool(),
           SaveTool(),
           UndoTool(),
           WheelZoomTool(),
       ]
       return tools

The ``HoverTool tooltips`` argument is a list of tuples - one tuple for each dimension of the data. The first argument of the tuple (e.g. "month") is the label that will appear when the user hover's over the data point, while the second (e.g. "@month_label") tells bokeh which column to use for the data (so it has to match the key you used in the ``ColumnDataSource`` creation).
       
Helper Functions
~~~~~~~~~~~~~~~~

The sub-figures needed some common elements so I created functions for them.

Scaling The Timestamps
++++++++++++++++++++++

The timestamps by default are unreadable (because there are so many). This re-scales them so they are more readable.

.. code:: python

   def scale_timestamp(index):
       """gets the scaled timestamp for element location
   
       Args:
        index: index in the portland.datetime series
       Returns:
        epoch timestamp used to locate place in plot
       """
       return portland.datetime[index].timestamp() * TIME_SCALE

Drawing the Recession
+++++++++++++++++++++

The recession is indicated as a blue box on each plot.

.. code:: python

   def make_recession():
       """Makes the box for the recession
   
       Returns:
        BoxAnnotation to color the recession
       """
       return BoxAnnotation(
           left=scale_timestamp(recession_start),
           right=scale_timestamp(recession_end),
           fill_color="blue",
           fill_alpha=0.1)   

Vertical Lines
++++++++++++++

Things like the unemployment lows and highs are indicated by a vertical line.

.. code:: python

   def make_vertical(location, color="darkorange"):
       """makes a vertical line
       
       Args:
        location: place on the x-axis for the line
        color (str): line-color for the line
       Returns:
        Span at index
       """
       return Span(
           location=location,
           line_color=color,
           dimension="height",
       )   

Make Verticals
++++++++++++++

Since there's more than one line, this function adds all the lines.

.. code:: python

   def make_verticals(fig):
       """makes the verticals and adds them to the figures"""
       fig.add_layout(make_vertical(
           location=scale_timestamp(unemployment_peaks[0]),
           color="darkorange",
       ))
       fig.add_layout(make_vertical(
           location=scale_timestamp(s_and_p_nadir[0]),
           color="crimson"))
       fig.add_layout(make_vertical(
           location=scale_timestamp(housing_nadir[0]),
           color="limegreen"))
       fig.add_layout(make_vertical(
           location=scale_timestamp(national_peak[0][0]),
           color="grey"))
       return
   

The Figures
~~~~~~~~~~~

This plot has three sub-figures, each of which is created separately then added to the ``Column``.

Unemployment
++++++++++++

.. code:: python
   
   tools = make_tools()
   unemployment_figure = figure(
       plot_width=FIGURE_WIDTH,
       plot_height=FIGURE_HEIGHT,
       x_axis_type="datetime",
       tools=tools,
       title="Portland Unemployment (2007-2017)"
   )

Next the lines for the time-series data are added.

.. code:: python

   unemployment_figure.line(
       "month_data", "unemployment",
       source=portland_source,
       line_color=PORTLAND_COLOR,
       legend=PORTLAND_LABEL,
             )
   
   line = unemployment_figure.line(
       "month_data", "unemployment",
       source=national_source,
       line_color=NATIONAL_COLOR,
       legend=NATIONAL_LABEL,
   )

Now the recession-box and high and low points for each plot is added.

.. code:: python

   unemployment_figure.add_layout(make_recession())
   make_verticals(unemployment_figure)

Now some labels are added and the grid is turned off.

.. code:: python
   
   unemployment_figure.yaxis.axis_label = "% Unemployment"
   unemployment_figure.xaxis.axis_label = "Month"
   unemployment_figure.xgrid.visible = False
   unemployment_figure.ygrid.visible = False

S & P 500
+++++++++

The S & P 500 had didn't have unemployment as the dependent variable so I made a different set of tools to change the label for the hover.

.. code:: python

   hover = HoverTool(tooltips=[
       ("Month", "@month_label"),
       ("Value", "@value"),
   ])
   tools = [
       hover,
       CrosshairTool(),
       PanTool(),
       ResetTool(),
       ResizeTool(),
       SaveTool(),
       UndoTool(),
       WheelZoomTool(),
   ]
   s_and_p_figure = figure(
       plot_width=FIGURE_WIDTH,
       plot_height=FIGURE_HEIGHT,
       x_range=unemployment_figure.x_range,
       x_axis_type="datetime",
       tools=tools,
       title="S & P 500 Index",
   )
   line = s_and_p_figure.line("month_data", "value",
                       source=s_and_p_source,
                       line_color=S_AND_P_COLOR)
   s_and_p_figure.add_layout(make_recession())
   make_verticals(s_and_p_figure)
   s_and_p_figure.yaxis.axis_label = "S & P 500 Valuation"
   s_and_p_figure.xaxis.axis_label = "Month"
   s_and_p_figure.xgrid.visible = False
   s_and_p_figure.ygrid.visible = False
   s_and_p_figure.legend.location = "bottom_right"

Housing
+++++++

.. code:: python
   
   hover = HoverTool(tooltips=[
       ("Month", "@month_label"),
       ("Price", "@price"),
   ])
   tools = [
       hover,
       CrosshairTool(),
       PanTool(),
       ResetTool(),
       ResizeTool(),
       SaveTool(),
       UndoTool(),
       WheelZoomTool(),
   ]
   housing_figure = figure(
       plot_width=FIGURE_WIDTH,
       plot_height=FIGURE_HEIGHT,
       x_range=unemployment_figure.x_range,
       x_axis_type="datetime",
       tools=tools,
       title="House Price Index",
   )
   line = housing_figure.line("month_data", "price",
                              source=housing_source,
                              line_color=HOUSING_COLOR)
   housing_figure.add_layout(make_recession())
   make_verticals(housing_figure)
   housing_figure.yaxis.axis_label = "Sale Price ($1,000)"
   housing_figure.xaxis.axis_label = "Month"
   housing_figure.xgrid.visible = False
   housing_figure.ygrid.visible = False
   housing_figure.legend.location = "bottom_right"

Combining
~~~~~~~~~

Once the figures were created I combined them into a ``column``, since I wanted them stacked verticallly.

.. code:: python

   combined = column(unemployment_figure, s_and_p_figure, housing_figure)

Outputting The Code
~~~~~~~~~~~~~~~~~~~

In order to be able to embed the code, you need to have bokeh export it. There are multiple ways to do this, but I chose the ``autoload_static`` method.

.. code:: python

   OUTPUT_JAVASCRIPT = "portland_unemployment.js"
   js, tag = autoload_static(combined, CDN, OUTPUT_JAVASCRIPT)

The third argument (``OUTPUT_JAVASCRIPT``) is the path you want to refer to in the tag. The returned ``js`` variable contains the javascript you need to save (using the filename you gave ``autoload_static``) and the ``tag`` contains the HTML tag that you embed to let the server know you want to use the javascript that was saved.

Since both values are just strings, and nothing was saved to disk, I saved it for later.   

.. code:: python

   with open(OUTPUT_JAVASCRIPT, "w") as writer:
       writer.write(js)
   
   with open("portland_tag.html", 'w') as writer:
       writer.write(tag)

Getting It Into Nikola
~~~~~~~~~~~~~~~~~~~~~~

The first thing was to create this file using ``nikola new_post`` (it's called bokeh-test.rst). Next I created a directory in the ``files`` folder that had the same name as this file (without the ".rst" extension) to put the javascript in so nikola would find it when I built the HTML.

.. code:: bash

   mkdir files/posts/bokeh-test

Once I copied the ``portland_unemployment.js`` file to the ``bokeh-test`` directory I opened the ``portland_tag.html`` file and embedded it directly into the post sing the ``raw`` restructureText directive.

.. code:: rst

   .. raw:: html
   
      <script
          src="portland_unemployment.js"
          id="686c5dd6-168a-4f7d-acbc-524875d93b59"
          data-bokeh-model-id="c473232a-dc2c-4b75-988c-f9bc6517b4b9"
          data-bokeh-doc-id="402d8e3c-1595-4d65-9f76-e11068c629ab"
      ></script>                   
