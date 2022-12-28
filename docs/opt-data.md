# Data Structures

## Pandas 

Pandas [documentation](https://pandas.pydata.org/pandas-docs/stable/user_guide/enhancingperf.html) lists ways to enhance the performances in terms of both query and storage.

### Storage & Memory

We can reduce the memory required to store the dataframes if we reduce the datatype to its minimal or appropriate ones. This can also make faster queries if categorical dtypes are assigned properly. 

```python
def reduce_df(df, cat_col=[], cat_percent=0.5, verbose=True):
    """"maximium data memory optimization
    
    Args:
        cat_col (list): list of columm names to convert to category dtype
        cat_percent (float): only convert if percentage duplicates more than stated
        verbose (bool): prints reduction percentage
    """
    mem_start = df.memory_usage().sum()

    for col in df.columns:
        dtype_ = df[col].dtype
        if dtype_ == float:
            df[col] = pd.to_numeric(df[col], downcast='float')
        elif dtype_ == int:
            min_ = df[col].min()
            if min_ > 0:
                df[col] = pd.to_numeric(df[col], downcast='unsigned')
            else:
                df[col] = pd.to_numeric(df[col], downcast='integer')
        elif dtype_ == object:
            if col in cat_col:
                dup_percent = 1 - (len(df[col].unique()) / size)
                if dup_percent >= cat_percent:
                    df[col] = df[col].astype('category')
    if verbose:
        mem_end = df.memory_usage().sum()
        print("{:.2f}% of original size is reduced".format((mem_start-mem_end) / mem_start * 100))
        
    return df
```

### Looping

As a rule of thumb, never use crude for loops to iterate and change values in the dataframe. The `apply` method is already internally optimized, like using iterators in Cython.

We can also used vectorization on pandas series or numpy arrays as suggested by this [author](https://engineering.upside.com/a-beginners-guide-to-optimizing-pandas-code-for-speed-c09ef2c6a4d6). 

```python
import numpy as np
import pandas as pd

# Define a basic Haversine distance formula
def haversine(lat1, lon1, lat2, lon2):
    MILES = 3959
    lat1, lon1, lat2, lon2 = map(np.deg2rad, [lat1, lon1, lat2, lon2])
    dlat = lat2 - lat1 
    dlon = lon2 - lon1 
    a = np.sin(dlat/2)**2 + np.cos(lat1) * np.cos(lat2) * np.sin(dlon/2)**2
    c = 2 * np.arcsin(np.sqrt(a)) 
    total_miles = MILES * c
    return total_miles

# optimized on apply-lambda
df['distance'] = df.apply(lambda row: haversine(40.671, -73.985, row['latitude'], row['longitude']), axis=1)
# optimized pandas series
df['distance'] = haversine(40.671, -73.985, df['latitude'], df['longitude'])
# optimized numpy array
df['distance'] = haversine(40.671, -73.985, df['latitude'].values, df['longitude'].values)
```

### Querying

In pandas, there are various ways of querying, and the speed various with each type. 

In the example below, the fastest way is to convert the query column into a category data-type as it contains many duplicates, hence conversion into integers underthehood make it query more efficient. However, this might not always be the case, and using `isin` or `in1d` might suffice.

More information [here](https://stackoverflow.com/questions/17071871/how-do-i-select-rows-from-a-dataframe-based-on-column-values).

```python
# Unoptimized, 0.01288s
result = df[df["antecedents"] == item]

# Optimized Queries
# -------------------
# with pandas .query, 0.00722s, x1.8
result = df.query("antecedents == @item")

# with isin, 0.00440s, x2.9
result = df[df["antecedents"].isin([item])]

# with indexing, 0.004342s x3.0
df = df.set_index("antecedents")
result = df[df.index.isin([item])]

# with numpy, 0.004175s, x3.1
result = df[np.in1d(df["antecedents"].values, [item])]

# using category datatype, 0.000608s, x211.8
df["antecedents"] = df["antecedents"].astype('category')
result = df[df["antecedents"] == item]
```

## Graph

A graph data structure enables much faster query if there is a network of interconnectivity between each other, known as **nodes**. Information stored between each connection of two nodes are called **links**. 

A simple graph structure in python can be created using the [networkx](https://networkx.org) library. However, this does not scale when the data become huge. For that, a graph database like [neo4j](https://neo4j.com) is more appropriate.

```python
import networkx as nx
import pandas as pd

df = pd.read_csv("association_rules.csv")
# convert dataframe to networkx graph
graph = nx.convert_matrix.from_pandas_edgelist(
            df, "antecedents", "consequents",
            [
                "antecedent support",
                "consequent support",
                "support",
                "confidence",
                "lift",
                "leverage",
                "conviction",
            ],
            create_using=nx.DiGraph,
        )
# query
result = list(graph[item])
```