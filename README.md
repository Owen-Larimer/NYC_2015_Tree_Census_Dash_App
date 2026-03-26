# NYC_2015_Tree_Census_Dash_App
A complete dash app utilizing plotly and DBC to allow users to explore the entire forest of trees living along all the streets of NYC.


This will be hosted in a public repository (Link Here: ...), deployed by Render or to Github Pages (TBD).

But in this Repository we will include the major skeleton of the probject and static versions of it.


The data we used for this project comes from data.gov, and is the 2015 NYC Tree Census set (CSV version). Link: [Dataset Menu](https://catalog.data.gov/dataset/2015-street-tree-census-tree-data-a16a1)

Additionally, you can find the metadata information page at the [City of New York's official data page](https://data.cityofnewyork.us/Environment/2015-Street-Tree-Census-Tree-Data/uvpi-gqnh/about_data). This page will tell you all about each variable/column.

Let's start by viewing our data in Python/Jupyter and getting a sense for its capabilities and what we want to do with it. We'll need a number of libraries for this whole project:

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import plotly.express as px
import dash
from dash import Dash, html, dash_table, dcc
from dash.dependencies import Input, Output
import dash_bootstrap_components as dbc
```

```python
nyc_trees = pd.read_csv("./2015_NYC_Tree_Census.csv")
nyc_trees.head()
```
<img width="2526" height="774" alt="image" src="https://github.com/user-attachments/assets/318a8cbc-cb06-4e4e-a7a9-16b5d36b5c33" />


Each row is a tree, and lists what Borough of NY it's in, it's diameter at breast height, the neighborhood it's in, any problems it has, and so on.

```python
nyc_trees.dtypes
```
<img width="417" height="1464" alt="image" src="https://github.com/user-attachments/assets/177d6a46-5cf2-4f59-8a34-2d383dca6ee1" />


One of the first things you may notice in this set is that we have numerical latitude and longitude columns. This means we can create a map of NY and plot the trees on their exact positions. This could be a great visualizer. Plotly is by far the easiest method for doing this. Since there is well over 650,000 trees in this set, let's make it a bit easier on our systems and take a random sample of 50,000 to plot, at least initially.

```python
map_df = nyc_trees.sample(50000, random_state=42)

fig = px.scatter_mapbox(
    map_df,
    lat="latitude", lon="longitude", color="health", size="tree_dbh", hover_name="spc_common", hover_data={
        "health": True,
        "tree_dbh": True,
        "borough": True,
        "latitude": False,
        "longitude": False
    }, zoom=10, height=700,
)

fig.update_layout(
    mapbox_style="carto-positron",
    margin={"r":0,"t":0,"l":0,"b":0},
)
```










