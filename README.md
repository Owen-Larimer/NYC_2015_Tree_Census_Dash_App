# NYC_2015_Tree_Census_Dash_App
A complete dash app utilizing plotly and DBC to allow users to explore the entire forest of trees living along all the streets of NYC.


This will be hosted in a public repository (Link Here: ...), deployed by Render or to Github Pages (TBD).

But in this Repository we will include the major skeleton of the probject and static versions of it.


The data we used for this project comes from data.gov, and is the 2015 NYC Tree Census set (CSV version). Link: [Dataset Menu](https://catalog.data.gov/dataset/2015-street-tree-census-tree-data-a16a1)

Additionally, you can find the metadata information page at the [City of New York's official data page](https://data.cityofnewyork.us/Environment/2015-Street-Tree-Census-Tree-Data/uvpi-gqnh/about_data). This page will tell you all about each variable/column.

Let's start by viewing our data in Python/Jupyter and getting a sense for its capabilities and what we want to do with it (See "Viewing_data.ipynb"). We'll need a number of libraries for this whole project:

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
<img width="2296" height="1410" alt="image" src="https://github.com/user-attachments/assets/2a088533-1eb9-4dcc-ba98-7c7f6cc80f58" />


Okay! It's looking solid. Due to plotly features, you can pan through it, zoom in, and when we hover over a tree on the map we get a little tooltip with the type of tree, it's health, it's dbh (Diameter at Breast Height), and the borough it's in. This will be useful for interactivity in our app later!

We can also do the same thing for each borough, as demonstrated with Queens below:


```python
queens_trees = nyc_trees[nyc_trees["borough"] == "Queens"]

queens_trees = queens_trees.sample(20000, random_state=42)


fig = px.scatter_mapbox(
    queens_trees,
    lat="latitude",
    lon="longitude",
    color="health",
    size="tree_dbh",
    hover_name="spc_common",
    hover_data={
        "borough": True,
        "tree_dbh": True,
        "health": True,
        "latitude": False,
        "longitude": False
    },
    zoom=11,
    height=700,
    color_discrete_map={
        "Good": "#2E8B57",
        "Fair": "#FFD700",
        "Poor": "#D62728"
    }
)

fig.update_layout(
    mapbox_style="carto-positron",
    margin=dict(l=0, r=0, t=0, b=0),
    title="Queens Street Trees"
)
```
<img width="2488" height="1367" alt="image" src="https://github.com/user-attachments/assets/994e9bec-ceb0-4936-8f1d-33f2a7e7dfc6" />

The perspective point of the map will change based upon the proposed latitude and longitude values automatically. This is a very useful feature of plotly and one that will make our final dash app a lot more user friendly. 

Queens is NYC's largest borough, and we can see that reflected in the amount of trees present:
```python
nyc_trees['borough'].value_counts()
```
<img width="415" height="252" alt="image" src="https://github.com/user-attachments/assets/d422466b-2c61-4e9b-bbee-3258bebae99d" />


Now, another feature of our set you may have noticed (and I've mentioned above) is the health of the trees. Each individual tree is given a health group determined from various features of the tree. We have "Good," "Fair," and "Poor." Now, those are helpful, but if we make them into numerics we can do a little bit more with them.

```python


# Map health: Good=3, Fair=2, Poor=1
health_map = {
    "Good": 3,
    "Fair": 2,
    "Poor": 1
}
nyc_trees["health_numeric"] = nyc_trees["health"].map(health_map)
```
<img width="810" height="415" alt="image" src="https://github.com/user-attachments/assets/a12d044f-f7e3-47d9-b11f-fa9e2cb2dcd9" />



Now we can really start subsetting our data, and creating interactions and relationships that we'll end up including in our final Dash app.


```python
grouped_boroughs = nyc_trees.groupby('borough')['tree_id'].count().reset_index()
grouped_boroughs
```
<img width="385" height="377" alt="image" src="https://github.com/user-attachments/assets/88a615f0-f12c-4e89-a457-72a89a884810" />


```python
nyc_trees[["borough", "health", "health_numeric"]].head()
```
<img width="555" height="383" alt="image" src="https://github.com/user-attachments/assets/db0358ac-e473-4e77-9720-9dc9086cb6f4" />


```python
nyc_trees['problems'].value_counts()
```
<img width="1234" height="467" alt="image" src="https://github.com/user-attachments/assets/7808d01d-c589-4760-83b2-7f5bdc550976" />


We can also subset our data by individual tree species instead of borough. This is very easy with a pivot table:
```python
selected_borough = "Queens"

if selected_borough == "All NYC":
    df = nyc_trees.copy()
else:
    df = nyc_trees[nyc_trees["borough"] == selected_borough]

species_health_counts = df.pivot_table(
    index="spc_common", columns="health", values="tree_id", aggfunc="count", fill_value=0  # fills missing combinations with 0
)

# Optional: sort species by total count
species_health_counts["total"] = species_health_counts.sum(axis=1)
species_health_counts = species_health_counts.sort_values(by="total", ascending=False).drop(columns="total")
species_health_counts.head()
```
<img width="505" height="432" alt="image" src="https://github.com/user-attachments/assets/b7009384-0ab7-4ec9-b8a7-f729fcf98344" />


But for a much more interesting visualization of this relationship we can use a heatmap, also available through plotly and it's px.density_heatmap method:

```python
df = nyc_trees.copy()

# Map health to a consistent order
health_order = ["Good", "Fair", "Poor"]
df["health"] = pd.Categorical(df["health"], categories=health_order, ordered=True)

# Find the 6 most common species in NYC
top_species = df["spc_common"].value_counts().nlargest(6).index.tolist()

# Subset to top species
df_top = df[df["spc_common"].isin(top_species)]

# Create a pivot table: rows=health, columns=species, values=count
heatmap_data = df_top.pivot_table(
    index="health",
    columns="spc_common",
    values="tree_id",
    aggfunc="count",
    fill_value=0
)

# Reset index for Plotly
heatmap_data = heatmap_data.reset_index()

# Melt into long format for plotly express
heatmap_melt = heatmap_data.melt(id_vars="health", var_name="Species", value_name="Count")

# Plotly heatmap
fig = px.density_heatmap(heatmap_melt, x="Species", y="health", z="Count",
    color_continuous_scale=["#C19A6B", "#90EE90", "#2E8B57"],  # poor->fair->good
    text_auto=True,
)

fig.update_layout(title="Tree Health Distribution for Top 6 NYC Species",
    xaxis_title="Species",
    yaxis_title="Health",
    yaxis=dict(categoryorder="array", categoryarray=health_order),
    height=500
)




fig.show()
```
<img width="2472" height="913" alt="image" src="https://github.com/user-attachments/assets/187ce8dd-5d7d-410c-899a-6ae047a78ff9" />

Beyond just being more visually appealing than the dataframe itself, we'll be able to connect figures like this to features of Dash like toggles, search-dropdowns, and sliders to subset our data in real time to reflect new relationships within the dash app.


Say we want to know the most common tree species in NYC. A bar chart might be the most beneficial for that:

```python
# Count the number of trees per species
species_counts = nyc_trees['spc_common'].value_counts()

# Select top 8 species
top_species = species_counts.head(8)

# Convert to a dataframe for plotting
top_species_df = top_species.reset_index()
top_species_df.columns = ['Species', 'Count']

# Make bar plot
fig = px.bar(top_species_df, y='Species', x='Count', text='Count', color='Count', color_continuous_scale='Greens', title='Most Common Tree Species in NYC', orientation='h')

# Add nicer formatting
fig.update_traces(texttemplate='%{text:,}', textposition='outside')
fig.update_layout(
    xaxis_title='Species',
    yaxis_title='Number of Trees',
    coloraxis_showscale=False,
    uniformtext_minsize=8,
    uniformtext_mode='hide'
)

fig.show()
```

<img width="2390" height="612" alt="image" src="https://github.com/user-attachments/assets/496d69d1-e712-4a5b-ba5a-3fc3f4ecffce" />
With figures such as bar chart, it may be beneficial to try both orientations to see which fits better in your end app, For now, we'll stick with horizontal. 



I'm thinking we'll want a section about tree health in our app. It's tough to say what the overall "healthiest" species is, as a tree is only given a score of 1-3, and problems with a tree are not necessarily indicative of poor health. Additionally, we must recognize that all our data was recorded by people investigating the trees, and thus is opened to potential bias or errors in recording. However, with so many individual trees, we can gain a collective sense of what trees seem to do well in an urban environment like NYC.

So, what *is* our healthiest species? That is, what kind of tree in NYC was the healthiest on average in 2015?

Let's start by subsetting only trees with 5 or more actual instances:
```python
# Copy your original dataframe to avoid modifying it
df = nyc_trees.copy()

# Only keep species with at least 10 instances
species_counts = df['spc_common'].value_counts()
species_eligible = species_counts[species_counts >= 5].index
df_filtered = df[df['spc_common'].isin(species_eligible)]

# Compute average health per species
avg_health_by_species = df_filtered.groupby('spc_common')['health_numeric'].mean()

# Find healthiest and unhealthiest species
healthiest_species = avg_health_by_species.idxmax()
unhealthiest_species = avg_health_by_species.idxmin()

print("Healthiest species:", healthiest_species)
print("Unhealthiest species:", unhealthiest_species)

# ------------------------------
# Pie chart for healthiest species
# ------------------------------
df_healthy = df_filtered[df_filtered['spc_common'] == healthiest_species]
health_counts_healthy = df_healthy['health'].value_counts().reindex(['Good','Fair','Poor']).fillna(0)

fig_healthiest = px.pie(
    names=health_counts_healthy.index,
    values=health_counts_healthy.values,
    title=f"Health Distribution of {healthiest_species} (Healthiest on Average)",
    color=health_counts_healthy.index,
    color_discrete_map={
        "Good": "#2E8B57",
        "Fair": "#90EE90",
        "Poor": "#C19A6B"
    }
)
fig_healthiest.show()

```

<img width="1293" height="642" alt="image" src="https://github.com/user-attachments/assets/d8e40b13-ad47-4265-845f-725e64cc57e9" />


And we can do the same with the unhealthiest species. That is, which species had the highest percentage of individuals given the "Poor" health tag?

```python
```
