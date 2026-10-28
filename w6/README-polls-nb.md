---
jupytext:
  custom_cell_magics: kql
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.19.5
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# US polls with altair

+++

```{admonition} download the zip
:class: warning

to run this code on your own laptop,
{download}`start with downloading the zip file<./ARTEFACTS-polls.zip>`
```

```{code-cell} ipython3
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

import geopandas as gpd

# activate this if running under jlab
# %matplotlib ipympl
```

## 1. loading presidential averages

538.com - actually <http://fivethirtyeight.com> - was a site that published data about the US presidential election  
the site is no longer online, so we use a copy of its **presidential averages** that is stored in `data/DATA.csv`

load this file into a dataframe `df`; make sure to
- be careful about the types of the columns, in particular `date`, that must be a proper timestamp
- notice that the first column is a leftover index, and so ensure it does not end up as a column in `df`

```{code-cell} ipython3
:tags: [level_basic]

# your code
df = pd.read_csv("data/DATA.csv")
df["date"] = pd.to_datetime(df["date"])
```

## 2. plotting

And what we want to do is to plot the average of the polls for each candidate.  
In other words, you should obtain something like this - we will arbitrarily focus on the 2024 year only

```{image} media/polls-over-time-2024.png
:width: 800px
:align: center
```

```{code-cell} ipython3
:tags: [level_basic]

# your code
df[df["date"].dt.year == 2024].groupby(["date","candidate"])['pct_estimate'].mean().unstack().plot()
```

## 3. use interactivity

using the interactive view (if under Jupyter Lab) make sure to use `%matplotlib ipympl`, and zoom into the figure and retrieve
- the date for the last data about Joe Biden
- the date for the first data about Kamala Harris

```{code-cell} ipython3
:tags: [level_basic]

# your answer in english - as a comment
#biden 2024-07-20
#harris 2024-07-26
%matplotlib ipympl
df_2024 = df[df["date"].dt.year == 2024]
fig, ax = plt.subplots(figsize=(14, 7))
for candidate, data in df_2024.groupby("candidate"):
    data = data.sort_values("date")
    ax.plot(data["date"], data["pct_estimate"], label=candidate)
plt.show()
```

also write a line of code to compute this second date

```{code-cell} ipython3
:tags: [level_basic]

harris = df[df["candidate"].str.contains("Harris")]
first_harris_date = harris["date"].min()
```

## 4. race end

from this part we will focus on the period after `first_harris_date`
1. how many candidates are still in the data ?
1. make sure to keep only the 2 most famous ones
1. how many data points do we have now for each of them ?

```{code-cell} ipython3
:tags: [level_basic]

# your code
df2 = df[df["date"] >= first_harris_date].copy()
print(df2["candidate"].nunique()) #3

print(df2["candidate"].value_counts()) #les plus famous sont kamala et trump

# 900 et 900
```

## 5: loading shapefiles with geopandas

```{code-cell} ipython3
import geopandas as gpd
```

we want to plot this data on a geographic map; first we need a definition of the various US states; this comes from the US Census Bureau (file `us-states.zip`), and we have a copy of it under `data/`

we load this shapefile into a geopandas object

```{code-cell} ipython3
gdf = gpd.read_file("data/us-states.zip")
```

1. how many entries ?

```{code-cell} ipython3
:tags: [level_basic]

# your code
print(len(gdf)) #52
print(gdf.dtypes)
```

2. what are the various column types

+++

as you can see, this is almost like a regular dataframe, except for the `geometry` column; which is a geographic entity, hence the term `geo-dataframe`

+++

## digression: using altair to produce a geographic viz

```{admonition} this may require an installation
:class: dropdown tip
of course you might have to install `altair`...  how do you go about doing that again ?
```

````{admonition} see also on altair and its use on geographic data
:class: dropdown warning

more details on this topic can be found here: 

- <https://altair-viz.github.io/user_guide/marks/geoshape.html>
- <https://idl.uw.edu/visualization-curriculum/altair_cartographic.html>
- <https://altair-viz.github.io/altair-tutorial/notebooks/09-Geographic-plots.html>
````

```{code-cell} ipython3
import altair as alt
alt.renderers.enable("html")
```

Here are a few examples of what we can do with this library

+++

### the simplest chart

```{code-cell} ipython3
alt.Chart(gdf).mark_geoshape()
chart = (
    alt.Chart(gdf)
    .mark_geoshape()
)

chart.display()
```

### improving that view

+++

now in terms of presentation, it is a little suboptimal, let's improve this a bit

```{code-cell} ipython3
(
    alt.Chart(gdf)
    .mark_geoshape()
    .properties(width=600)
    .project('albersUsa')
)
```

### using colors

+++

now, the initial geo-dataframe has some numeric values, that we can use to color the map !  
for example, there are `AWATER` and `ALAND`  
which we can interpret as meaning *area of water* (as in lakes) and *area of land* respectively

```{code-cell} ipython3
gdf.head(2)
```

and we can use one of these to color the different states  
for that we would use `encode()` like so

```{code-cell} ipython3
(
    alt.Chart(gdf)
    .mark_geoshape()
    .encode(
        color="ALAND:Q",          # Q stands for quantitative
    )
    .properties(width=600)
    .project('albersUsa')
)
```

### better with a legend

```{code-cell} ipython3
# or if we prefer, same result essentially
# but this way we can be more descriptive

(
    alt.Chart(gdf)
    .mark_geoshape()
    # legend
    .encode(
        color=alt.Color(field="ALAND", type="quantitative", title="land area")
    )
    # a bit narrower so the legend shows up
    .properties(width=600)
    .project('albersUsa')
)
```

### even better with tooltips

```{code-cell} ipython3
# also useful with altair, we can give a `tooltip` parameter to encode
# and this shows when your mouse hovers on a state

# NOTE: altair can only see the columns, not the index
# so as we want to show the name of the state, we need it in a regular column
# so you had made the choice to use the state as the index
# you may need to reset_index()

(
    alt.Chart(gdf)
    .mark_geoshape()
    .encode(
        color=alt.Color(field="AWATER", type="quantitative", title="water area"),
        # and we can show there anything from the table
        tooltip=["NAME", "ALAND"],
    )
    .properties(width=600)
    .project('albersUsa')
)
```

## our goal: geographic rendering

in this following we will produce a summary map, which looks like this:
- for all the states where we have data
- the color depicts the ratio between, Harris's and Trump's average scores over time
- also the tooltips allow to expose more details on the individual results

```{image} media/states-with-data.png
:width: 800px
:align: center
```

+++

## 6. first map

+++

1. for how many states do we have polls data ? which are they ?

```{code-cell} ipython3
:tags: [level_basic]

# your code
len(harris.state.value_counts()) #21
```

2. for this data, compute a table with the state as index, and the two contestants polls (averaged)

```{code-cell} ipython3
:tags: [level_basic]

# your code
states_with_data = sorted(df2.loc[df2["state"].notna()&(df2["state"] != "US"),"state"].unique())
state_polls = (df2[df2["state"].isin(states_with_data)&df2["candidate"].isin(most_famous)].groupby(["state", "candidate"])["pct_estimate"].mean().unstack())
print(state_polls.head())
```

3. using what you've seen in the digression, and what you will gather if needed on the Internet,  produce the above map

```{code-cell} ipython3
:tags: [level_basic]

# your code
#to_draw=pivot.merge(gdf, left_index=True, right_on="NAME")
#(to_draw.Chart(gdf).mark_geoshape().encode().properties(width=600).project('albertUsa')) non abouti
```

## 7. outlining swing states

+++

from the graph above, we want to outline the swing states  
for example, we could come up with something like

```{image} media/swing-states-outlined.png
:width: 800px
:align: center
```

````{admonition} *hints*
:class: dropdown

- the method `pd.Series.isin()` might come in handy for this step
- also you will probably need to search a bit by yourself, outside of what we've covered in the digression
````

```{code-cell} ipython3
SWING_STATES = [
    'Nevada',
    'Arizona',
    'Wisconsin',
    'Michigan',
    'Pennsylvania',
    'Georgia',
    'North Carolina',
]
```

```{code-cell} ipython3
:tags: [level_basic]

# your code
map_df = gdf.reset_index(drop=True).copy()
polls_for_map = state_polls.reset_index()
map_df = map_df.merge(polls_for_map,left_on="NAME",right_on="state",how="left")
map_df["margin"] = (map_df["Harris"] - map_df["Trump"])
base_map = (alt.Chart(map_df).mark_geoshape().encode(color=alt.Color("margin:Q")))
swing_outline = (alt.Chart(map_df).transform_filter(alt.FieldOneOfPredicate(field="NAME",oneOf=SWING_STATES)).mark_geoshape(stroke="black",strokeWidth=2))
chart = ((base_map + swing_outline))
chart.display()
```

## 8. filling gaps (optional)

this is not very readable; instead we want to still get the missing states show up in gray,  
and to keep the outlining of the swing states, i.e.

```{image} media/all-states.png
:width: 800px
:align: center
```

```{admonition} tip
:class: dropdown tip
this one is less obvious, and you are allowed to use an IA to find the solution !
```

```{code-cell} ipython3
:tags: [level_basic]

# your code
```
