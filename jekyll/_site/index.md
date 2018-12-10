
# A time Story of offshore Societies

You can see the aim of this project in the README file.

Our work is based on 4 databases, namely the *bahamas leaks*, the *panama papers*, the *offshore leaks* and the *paradise papers*. They all contain csv files with all the data of a graph : nodes (one file for each type of node), and edges. We merged all these files in this notebook.

You will see all the steps of our data exploration. In the end our objective is to arrive to two Dataframe, one containing all the nodes and the relevant information, and one containing the edges with relevant information.

## Imports


```python
import pandas as pd
import numpy as np
import networkx as nx #For graphs
import copy
import warnings
from datetime import *
import dateutil.parser
warnings.filterwarnings("ignore")

import matplotlib.pyplot as plt

# https://www.occrp.org/en/panamapapers/database
# TRUMP OFFSHORE INC. is good example to see all entities interacting
```

## Filenames / paths

The data is separated for every leak source. For each leak source there is a folder containing the nodes of the graph, that can be of different types : <i>intermediary, officer, entity, address</i> (and <i>other</i> for paradise papers only). The folder also contains the edges of this graph.


```python
bahamas_folder = "bahamas/"
panama_folder = "panama/"
paradise_folder = "paradise/"
offshore_folder = "offshore/"

sources_names = ['bahamas', 'panama', 'paradise', 'offshore']

panama_name = panama_folder + "panama_papers"
paradise_name = paradise_folder + "paradise_papers"
offshore_name = offshore_folder + "offshore_leaks"
bahamas_name = bahamas_folder + "bahamas_leaks"

edges_name = ".edges"
nodes_name = ".nodes."

address_name = "address"
intermediary_name = "intermediary"
officer_name = "officer"
entity_name = "entity"
others_name = "other" # Only for paradise paper there is this extra entity

usual_entity_names = [address_name, intermediary_name, officer_name, entity_name]
```

## Build local storage

We store data in dictionnaries that map each leak source to its content, which is a dictionnary that maps each type of entity to the Dataframe containing its values. For example <b>d_sources["bahamas"]["officer"]</b> is the Dataframe of officers coming from the bahamas leaks.


```python
def my_read_csv(filename) :
    """ To have same rules when reading data from csv """
    return pd.read_csv(filename, dtype = str)

def build_dict(source_name):
    """
    Create a dictionnary for a certain source_name (among : Panama papers, Paradise papers...)
    that maps to each entity name (among : Officer, Intermediary, Address...)
    the content of the csv from source_name for this entity
    """
    d = {en : my_read_csv(source_name + nodes_name + en + ".csv") for en in usual_entity_names}
    
    if source_name == paradise_name: # Extra "other" entity in paradise papers
        d[others_name] = my_read_csv(source_name + nodes_name + others_name + ".csv")
    
    #Add edges
    d["edges"] = my_read_csv(source_name + edges_name + ".csv")
              
    return d
```

Build the dictionnary, that maps each source to its content


```python
d_sources = dict()
d_sources["bahamas"] = build_dict(bahamas_name)
d_sources["panama"] = build_dict(panama_name)
d_sources["paradise"] = build_dict(paradise_name)
d_sources["offshore"] = build_dict(offshore_name)
```


```python
d_sources['panama']['entity'].columns
```




    Index(['node_id', 'name', 'jurisdiction', 'jurisdiction_description',
           'country_codes', 'countries', 'incorporation_date', 'inactivation_date',
           'struck_off_date', 'closed_date', 'ibcRUC', 'status', 'company_type',
           'service_provider', 'sourceID', 'valid_until', 'note'],
          dtype='object')



## Getting familiar with the data format

### Define some coloring for printing

Keep the same coloring during the project, it makes data very easily readable once you get familiar with the coloring !


```python
BOLD = '\033[1m'
BLUE = '\033[94m'
GREEN = '\033[92m'
YELLOW = '\033[93m'
RED = '\033[91m'
END = '\033[0m'

color_dict = dict()
color_dict["bahamas"] = YELLOW
color_dict["paradise"] = GREEN
color_dict["panama"] = RED
color_dict["offshore"] = BLUE

def color(str):
    """
    Returns the str given in the color of the source it is from 
    (the str must contain source name)
    """
    for source in color_dict.keys():
        if source in str:
            return color_dict[source] + str + END 
        
    return BOLD + str + END #Default color is BOLD

for name, _ in color_dict.items():
    print(color(name))
print(color("Unknown source"))
```

    [93mbahamas[0m
    [92mparadise[0m
    [91mpanama[0m
    [94moffshore[0m
    [1mUnknown source[0m


### See what data source misses which column


```python
for source, dict_data in d_sources.items():
    for source_compare, dict_data_compare in d_sources.items():
        print("\n", color(source_compare), "missing columns from source :", color(source))
        for entity in usual_entity_names:
            missing_columns = []
            for col in dict_data[entity].columns:
                if not col in dict_data_compare[entity].columns:
                    missing_columns.append(col)
            if(len(missing_columns) > 0):
                print("Node type", entity, "misses", len(missing_columns), "columns, namely : ", missing_columns)

```

    
     [93mbahamas[0m missing columns from source : [93mbahamas[0m
    
     [91mpanama[0m missing columns from source : [93mbahamas[0m
    Node type address misses 10 columns, namely :  ['labels(n)', 'jurisdiction_description', 'service_provider', 'jurisdiction', 'closed_date', 'incorporation_date', 'ibcRUC', 'type', 'status', 'company_type']
    Node type intermediary misses 10 columns, namely :  ['labels(n)', 'address', 'jurisdiction_description', 'service_provider', 'jurisdiction', 'closed_date', 'incorporation_date', 'ibcRUC', 'type', 'company_type']
    Node type officer misses 11 columns, namely :  ['labels(n)', 'address', 'jurisdiction_description', 'service_provider', 'jurisdiction', 'closed_date', 'incorporation_date', 'ibcRUC', 'type', 'status', 'company_type']
    Node type entity misses 3 columns, namely :  ['labels(n)', 'address', 'type']
    
     [92mparadise[0m missing columns from source : [93mbahamas[0m
    Node type address misses 10 columns, namely :  ['labels(n)', 'jurisdiction_description', 'service_provider', 'jurisdiction', 'closed_date', 'incorporation_date', 'ibcRUC', 'type', 'status', 'company_type']
    Node type intermediary misses 11 columns, namely :  ['labels(n)', 'address', 'jurisdiction_description', 'service_provider', 'jurisdiction', 'closed_date', 'incorporation_date', 'ibcRUC', 'type', 'status', 'company_type']
    Node type officer misses 10 columns, namely :  ['labels(n)', 'address', 'jurisdiction_description', 'service_provider', 'jurisdiction', 'closed_date', 'incorporation_date', 'ibcRUC', 'type', 'company_type']
    Node type entity misses 3 columns, namely :  ['labels(n)', 'address', 'type']
    
     [94moffshore[0m missing columns from source : [93mbahamas[0m
    Node type address misses 10 columns, namely :  ['labels(n)', 'jurisdiction_description', 'service_provider', 'jurisdiction', 'closed_date', 'incorporation_date', 'ibcRUC', 'type', 'status', 'company_type']
    Node type intermediary misses 10 columns, namely :  ['labels(n)', 'address', 'jurisdiction_description', 'service_provider', 'jurisdiction', 'closed_date', 'incorporation_date', 'ibcRUC', 'type', 'company_type']
    Node type officer misses 11 columns, namely :  ['labels(n)', 'address', 'jurisdiction_description', 'service_provider', 'jurisdiction', 'closed_date', 'incorporation_date', 'ibcRUC', 'type', 'status', 'company_type']
    Node type entity misses 3 columns, namely :  ['labels(n)', 'address', 'type']
    
     [93mbahamas[0m missing columns from source : [91mpanama[0m
    Node type entity misses 2 columns, namely :  ['inactivation_date', 'struck_off_date']
    
     [91mpanama[0m missing columns from source : [91mpanama[0m
    
     [92mparadise[0m missing columns from source : [91mpanama[0m
    Node type intermediary misses 1 columns, namely :  ['status']
    
     [94moffshore[0m missing columns from source : [91mpanama[0m
    
     [93mbahamas[0m missing columns from source : [92mparadise[0m
    Node type entity misses 2 columns, namely :  ['inactivation_date', 'struck_off_date']
    
     [91mpanama[0m missing columns from source : [92mparadise[0m
    Node type officer misses 1 columns, namely :  ['status']
    
     [92mparadise[0m missing columns from source : [92mparadise[0m
    
     [94moffshore[0m missing columns from source : [92mparadise[0m
    Node type officer misses 1 columns, namely :  ['status']
    
     [93mbahamas[0m missing columns from source : [94moffshore[0m
    Node type entity misses 2 columns, namely :  ['inactivation_date', 'struck_off_date']
    
     [91mpanama[0m missing columns from source : [94moffshore[0m
    
     [92mparadise[0m missing columns from source : [94moffshore[0m
    Node type intermediary misses 1 columns, namely :  ['status']
    
     [94moffshore[0m missing columns from source : [94moffshore[0m


We see that <span style="color:orange">bahamas</span> is the most "complete" source, in the sense it is the one that has the biggest number of columns missing in the others. We will therefore use it to explore the content of columns. *'inactivation_date'* and  *'struck_off_date'* columns from entity will then be explored in <span style="color:red">panama</span>

#### Special case : Paradise paper, <i>other</i> node


```python
d_sources["paradise"]["other"].columns
```




    Index(['node_id', 'name', 'country_codes', 'countries', 'sourceID',
           'valid_until', 'note'],
          dtype='object')



### SourceID in different sources

We see paradise papers is the only source that has different sourceID


```python
for source, dict_data in d_sources.items():
    print("\nSource :", color(source))
    for entity in usual_entity_names:
        value_count =  dict_data[entity]["sourceID"].value_counts()
        print("Node :", entity, len(value_count), "different sourceID :")
```

    
    Source : [93mbahamas[0m
    Node : address 1 different sourceID :
    Node : intermediary 1 different sourceID :
    Node : officer 1 different sourceID :
    Node : entity 1 different sourceID :
    
    Source : [91mpanama[0m
    Node : address 1 different sourceID :
    Node : intermediary 1 different sourceID :
    Node : officer 1 different sourceID :
    Node : entity 1 different sourceID :
    
    Source : [92mparadise[0m
    Node : address 7 different sourceID :
    Node : intermediary 5 different sourceID :
    Node : officer 9 different sourceID :
    Node : entity 9 different sourceID :
    
    Source : [94moffshore[0m
    Node : address 1 different sourceID :
    Node : intermediary 1 different sourceID :
    Node : officer 1 different sourceID :
    Node : entity 1 different sourceID :


### Check if node_id is a good index for Nodes


```python
merged_node_id = pd.Series()

for source, dict_data in d_sources.items():
    merged_node_id_source = pd.Series()
    for entity in usual_entity_names:
        
        merged_node_id_source = merged_node_id_source.append(dict_data[entity]["node_id"], ignore_index = True)
        
        if not dict_data[entity]["node_id"].is_unique:
            print("node_id isn't unique for source", color(source, "node", entity))
                  
    if not merged_node_id_source.is_unique:
        print("node_id isn't unique between nodes from source", color(source))
    
    merged_node_id = merged_node_id.append(merged_node_id_source.drop_duplicates())

if merged_node_id.is_unique:
    print("node_id is unique between unique nodes from all sources")
```

    node_id isn't unique between nodes from source [94moffshore[0m
    node_id is unique between unique nodes from all sources


So for each node type indepently node_id is a good index. Therefore (node_id, node_type) could be a good index (node_type being amond officer, intermediary...)

Now explore nodes with same node_id in offshore


```python
for i in range(len(usual_entity_names)):
    for j in range(i+1, len(usual_entity_names)):

        left_node = usual_entity_names[i]
        node = usual_entity_names[j]
        print(color(left_node), color(node))
        
        if left_node != node:

            left = d_sources["offshore"][left_node].set_index("node_id")
            right = d_sources["offshore"][node].set_index("node_id")

            intersection = left.join(right, on = "node_id", how = 'inner', \
                                     lsuffix = "_" + left_node,rsuffix = "_" + node)

            if not intersection.empty:
                print("Intersection of", color(left_node), "and", color(node), "count is :")
                print(intersection.count())
```

    [1maddress[0m [1mintermediary[0m
    [1maddress[0m [1mofficer[0m
    [1maddress[0m [1mentity[0m
    [1mintermediary[0m [1mofficer[0m
    Intersection of [1mintermediary[0m and [1mofficer[0m count is :
    name_intermediary             1139
    country_codes_intermediary    1139
    countries_intermediary        1139
    status                           0
    sourceID_intermediary         1139
    valid_until_intermediary      1139
    note_intermediary                0
    name_officer                  1139
    country_codes_officer         1139
    countries_officer             1139
    sourceID_officer              1139
    valid_until_officer           1139
    note_officer                     0
    dtype: int64
    [1mintermediary[0m [1mentity[0m
    [1mofficer[0m [1mentity[0m


So the intersection on offshore is between officer and intermediary nodes. Let's see if they are the same values :


```python
left = d_sources["offshore"]["officer"].set_index("node_id")
right = d_sources["offshore"]["intermediary"].set_index("node_id")

intersection = left.join(right, on = "node_id", how = 'inner', lsuffix = "_officer",rsuffix = "_interm")

intersection.loc[intersection["name_officer"] != intersection["name_interm"]].empty
```




    True



Therefore we understand that if someone appears in two different node types, it means it is the same person, but has two roles. This is why in further analysis we will store the pair (node_id, role) as index, because it is unique. We have to add a column to nodes, containing the node type, let's call it label. We saw in the column exploration that bahamas has an equivalent column *labels(n)*, that the other's don't, we'll rename it to *label*

## Keep necessary columns, Shape data to our need


```python
d_clean = dict()

#maps every node type to the columns to keep
d_columns = dict()
d_columns['address'] = ['country_codes', 'node_id']
d_columns['entity'] = ['node_id','name','jurisdiction','incorporation_date']
d_columns['intermediary'] = ['node_id', 'country_codes','name']
d_columns['officer'] = ['node_id', 'country_codes','name']
d_columns['other'] = ['node_id', 'country_codes','name']


for source, d in d_sources.items():
    
    d_clean[source] = dict()
    
    for node_type in usual_entity_names:
        d_clean[source][node_type] = d[node_type][d_columns[node_type]]
        d_clean[source][node_type]['source'] = source
        d_clean[source][node_type]['type'] = node_type
        d_clean[source][node_type]['node_id'] = d_clean[source][node_type]['node_id'].astype(np.int32)
    
    columns_edges = ['START_ID', 'END_ID', 'TYPE', 'start_date', 'end_date']        
    columns_edges_bahamas = ['node_1', 'node_2', 'rel_type', 'start_date', 'end_date']    
    
    if source == "bahamas": # adapt different column names
        d_clean[source]['edges'] = d_sources[source]['edges'][columns_edges_bahamas]
        d_clean[source]['edges'].columns = columns_edges
        
    else :
        d_clean[source]['edges'] = d_sources[source]['edges'][columns_edges]
    
    d_clean[source]['edges']['source'] = source
    d_clean[source]['edges']["START_ID"] = d_clean[source]['edges']["START_ID"].astype(np.int32)
    d_clean[source]['edges']["END_ID"] = d_clean[source]['edges']["END_ID"].astype(np.int32) 

d_clean['paradise']['other'] = d_sources['paradise']['other'][d_columns['other']]
d_clean["paradise"]['other']['source'] = 'paradise'
d_clean["paradise"]['other']['type'] = 'other'
```

### Create dictionaries for countries and jurisdictions

These dictionaries map the abrevation of countries to their full name, this way we can drop the longer column


```python
countries = dict()
jurisdictions = dict()

for s in sources_names:
    for t in usual_entity_names:
        countries.update(dict(zip(d_sources[s][t]['country_codes'], d_sources[s][t]['countries'])))
        if t  == 'entity':
            jurisdictions.update(dict(zip(d_sources[s][t]['jurisdiction'], d_sources[s][t]['jurisdiction_description'])))
            
countries.update(dict(zip(d_sources['paradise']['other']['country_codes'],\
                          d_sources['paradise']['other']['countries'])))
```

## Create and study *node* dataframe

A pourcentage function to print pourcentages in a nice way


```python
def pourcentage(n, precision = 2):
    """ To print a pourcentage in a nice way and with a given precision"""
    return color(("%." + str(precision) + "f") % (100.0*n) + "%")
```

A function to convert string of date to datetime format. There are a LOT of different string formats and we cover most of them. Dates with ambiguity such as 01/03/2001 are treated arbitrarily. (i.e. is this date 1st of March or 3rd of January ?) Indeed the year is generally what matters the most for us.

Years are valid until 2015 at most, and starting in the 1960s according to wikipedia (https://en.wikipedia.org/wiki/Panama_Papers)

When date is clearly an outlier (18/19/2006), it is set to NaN, and printed


```python
def parse_date(date):
    """ Parsing of the date, read above for more details"""
    if (date==date):
        try:
            formatted = dateutil.parser.parse(date)
            if (formatted.year > 2015 or formatted.year < 1960):
                formatted = 'NaN'
            return formatted 
        except:
            print(date)
            return 'NaN'
```

### Node types that should contain NaN for each column name

Nodes can have NaN values because of missing data, <b>or</b> because the data doesn't make sense for this node type. You will here find a list of node types for each column, those are node types that are NaN because of this second reason. For example the jurisdiction for an Officer doesn't really make sense at first sight... We will however in the future try to cumpute all the jurisdictions an officer is related to using the edges (and many more)

##### name
- Address

##### jurisdiction and incorporation_date
- Officer
- Other
- Intermediary
- Address

##### country_codes
- Entity



```python
nodes = pd.DataFrame(columns=['node_id','source','type','name','country_codes', 'jurisdiction', 'incorporation_date'])

for source,_ in d_sources.items():
    for node_type in usual_entity_names:
        nodes = nodes.append(d_clean[source][node_type], sort=False)
        
nodes = nodes.append(d_clean['paradise']['other'], sort=False)

nodes = nodes.set_index(['node_id', 'type'])

nodes['incorporation_date'] = nodes['incorporation_date'].apply(parse_date)
nodes['incorporation_date'] = pd.to_datetime(nodes['incorporation_date'])
```


```python
nodes.describe()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>source</th>
      <th>name</th>
      <th>country_codes</th>
      <th>jurisdiction</th>
      <th>incorporation_date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>1909605</td>
      <td>1534558</td>
      <td>697326</td>
      <td>785124</td>
      <td>756095</td>
    </tr>
    <tr>
      <th>unique</th>
      <td>4</td>
      <td>1252082</td>
      <td>3573</td>
      <td>80</td>
      <td>15558</td>
    </tr>
    <tr>
      <th>top</th>
      <td>paradise</td>
      <td>THE BEARER</td>
      <td>CHN</td>
      <td>BAH</td>
      <td>1998-01-02 00:00:00</td>
    </tr>
    <tr>
      <th>freq</th>
      <td>867931</td>
      <td>70871</td>
      <td>69394</td>
      <td>209634</td>
      <td>1368</td>
    </tr>
    <tr>
      <th>first</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>1960-01-01 00:00:00</td>
    </tr>
    <tr>
      <th>last</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>2015-12-31 00:00:00</td>
    </tr>
  </tbody>
</table>
</div>



It looks like there are a lot of unique country_codes... Indeed we notice some nodes have many country codes separated by a ';'


```python
country_codes = nodes.country_codes.dropna()
number_multi_country = country_codes[country_codes.str.contains(";")].count()

print(pourcentage(number_multi_country/len(country_codes)), "of nodes with a country_code have a country_code with more than one country")
```

    [1m4.02%[0m of nodes with a country_code have a country_code with more than one country


### Study by node type


```python
nodes.xs('entity', level = 1).describe()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>source</th>
      <th>name</th>
      <th>country_codes</th>
      <th>jurisdiction</th>
      <th>incorporation_date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>785124</td>
      <td>785095</td>
      <td>0.0</td>
      <td>785124</td>
      <td>756095</td>
    </tr>
    <tr>
      <th>unique</th>
      <td>4</td>
      <td>754951</td>
      <td>0.0</td>
      <td>80</td>
      <td>15558</td>
    </tr>
    <tr>
      <th>top</th>
      <td>paradise</td>
      <td>DE PALM TOURS N.V. - DE PALM TOURS</td>
      <td>NaN</td>
      <td>BAH</td>
      <td>1998-01-02 00:00:00</td>
    </tr>
    <tr>
      <th>freq</th>
      <td>290086</td>
      <td>19</td>
      <td>NaN</td>
      <td>209634</td>
      <td>1368</td>
    </tr>
    <tr>
      <th>first</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>1960-01-01 00:00:00</td>
    </tr>
    <tr>
      <th>last</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>2015-12-31 00:00:00</td>
    </tr>
  </tbody>
</table>
</div>




```python
nodes.xs('officer', level = 1).describe()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>source</th>
      <th>name</th>
      <th>country_codes</th>
      <th>jurisdiction</th>
      <th>incorporation_date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>720862</td>
      <td>720800</td>
      <td>423683</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>unique</th>
      <td>4</td>
      <td>507696</td>
      <td>3510</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>top</th>
      <td>paradise</td>
      <td>THE BEARER</td>
      <td>MLT</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>freq</th>
      <td>350008</td>
      <td>70871</td>
      <td>44916</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>




```python
nodes.xs('intermediary', level = 1).describe()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>source</th>
      <th>name</th>
      <th>country_codes</th>
      <th>jurisdiction</th>
      <th>incorporation_date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>25745</td>
      <td>25744</td>
      <td>23152</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>unique</th>
      <td>4</td>
      <td>24621</td>
      <td>285</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>top</th>
      <td>panama</td>
      <td>HUTCHINSON GAYLE A.</td>
      <td>HKG</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>freq</th>
      <td>14110</td>
      <td>62</td>
      <td>4895</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>




```python
nodes.xs('address', level = 1).describe()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>source</th>
      <th>name</th>
      <th>country_codes</th>
      <th>jurisdiction</th>
      <th>incorporation_date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>374955</td>
      <td>0.0</td>
      <td>250105</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>unique</th>
      <td>4</td>
      <td>0.0</td>
      <td>216</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>top</th>
      <td>paradise</td>
      <td>NaN</td>
      <td>CHN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>freq</th>
      <td>223350</td>
      <td>NaN</td>
      <td>31984</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>




```python
nodes.xs('other', level = 1).describe()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>source</th>
      <th>name</th>
      <th>country_codes</th>
      <th>jurisdiction</th>
      <th>incorporation_date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>2919</td>
      <td>2919</td>
      <td>386</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>unique</th>
      <td>1</td>
      <td>2912</td>
      <td>63</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>top</th>
      <td>paradise</td>
      <td>ANTAM ENTERPRISES N.V.</td>
      <td>IMN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>freq</th>
      <td>2919</td>
      <td>2</td>
      <td>105</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>



## Create and study *edges* dataframe


```python
edges = pd.DataFrame(columns=['START_ID', 'END_ID', 'TYPE', 'start_date', 'end_date','source'])
for source in sources_names:
    edges = edges.append(d_clean[source]['edges'], sort=False)

edges['start_date'] = pd.to_datetime(edges['start_date'].apply(parse_date))
edges['end_date'] = pd.to_datetime(edges['end_date'].apply(parse_date))
```

    18/19/2015
    16/20/2013
    05152015
    05152015
    05152015
    05152015
    05152015
    05152015
    05152015
    05152015
    1212012
    29/02/2006
    29/02/2006
    28.02/2014
    06/01.2009
    25090015
    31/02/2013
    31/02/2013
    31/02/2013
    12-112013
    284/2015


Printed strings are dates that were not read correctly


```python
edges.describe()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>START_ID</th>
      <th>END_ID</th>
      <th>TYPE</th>
      <th>start_date</th>
      <th>end_date</th>
      <th>source</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>3142523.0</td>
      <td>3142523.0</td>
      <td>3142523</td>
      <td>928670</td>
      <td>266893</td>
      <td>3142523</td>
    </tr>
    <tr>
      <th>unique</th>
      <td>1054794.0</td>
      <td>1224551.0</td>
      <td>13</td>
      <td>12353</td>
      <td>8545</td>
      <td>4</td>
    </tr>
    <tr>
      <th>top</th>
      <td>54662.0</td>
      <td>236724.0</td>
      <td>officer_of</td>
      <td>1969-12-31 00:00:00</td>
      <td>2012-09-30 00:00:00</td>
      <td>paradise</td>
    </tr>
    <tr>
      <th>freq</th>
      <td>36373.0</td>
      <td>37338.0</td>
      <td>1667166</td>
      <td>1848</td>
      <td>9663</td>
      <td>1657838</td>
    </tr>
    <tr>
      <th>first</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>1960-03-10 00:00:00</td>
      <td>1960-05-30 00:00:00</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>last</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>2015-12-31 00:00:00</td>
      <td>2015-12-31 00:00:00</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>



We see there are <b>13</b> unique kind of edges, they are listed below. In further analysis it will be interesting to study each of them in more depth.


```python
edges.TYPE.unique()
```




    array(['same_address_as', 'same_company_as', 'similar_company_as',
           'intermediary_of', 'registered_address', 'same_name_as',
           'same_intermediary_as', 'officer_of', 'probably_same_officer_as',
           'connected_to', 'same_id_as', 'same_as', 'underlying'],
          dtype=object)



### Study dates


```python
number_edges = len(edges)
number_start_date = edges.start_date.notna().sum()
number_end_date = edges.end_date.notna().sum()
```


```python
print(pourcentage(number_start_date/number_edges), "of edges have a start date")
print("Among those,", pourcentage(number_end_date/(number_end_date+number_start_date)), "have an end date")
print("The first added relation was the", edges.start_date.min())
print("The last added relation was the", edges.start_date.max())
```

    [1m29.55%[0m of edges have a start date
    Among those, [1m22.32%[0m have an end date
    The first added relation was the 1960-03-10 00:00:00
    The last added relation was the 2015-12-31 00:00:00


### Study average and extreme values


```python
print(edges.START_ID.value_counts().describe(), '\n')
print(edges.END_ID.value_counts().describe())
```

    count    1.054794e+06
    mean     2.979277e+00
    std      6.123411e+01
    min      1.000000e+00
    25%      1.000000e+00
    50%      1.000000e+00
    75%      2.000000e+00
    max      3.637300e+04
    Name: START_ID, dtype: float64 
    
    count    1.224551e+06
    mean     2.566266e+00
    std      3.778810e+01
    min      1.000000e+00
    25%      1.000000e+00
    50%      1.000000e+00
    75%      2.000000e+00
    max      3.733800e+04
    Name: END_ID, dtype: float64


We see that on average <b>2.98</b> edges start with the same node_id, and <b>2.57</b> end with the same node_id

Max number of connections starting from a given node is <b>36373</b>, and <b>37338</b> ending (another node)

## Study connection between edges and nodes


```python
id_max_links_start = edges.START_ID.value_counts().idxmax()
max_links_start = edges.START_ID.value_counts().max()
node_max_links_start = nodes.xs(id_max_links_start, level=0)

id_max_links_end = edges.END_ID.value_counts().idxmax()
max_links_end = edges.END_ID.value_counts().max()
node_max_links_end = nodes.xs(id_max_links_end, level=0)
```


```python
print("The Node with the most START edges has", color(str(max_links_start)), "links and is :")
print(node_max_links_start[['source', 'name', 'country_codes']])

print("\nThe Node with the most END edges has", color(str(max_links_end)), "links and is :")
print(node_max_links_end[['source', 'country_codes']])
```

    The Node with the most START edges has [1m36373[0m links and is :
                    source                               name    country_codes
    type                                                                      
    intermediary  offshore  Portcullis TrustNet (BVI) Limited  THA;VGB;IDN;SGP
    officer       offshore  Portcullis TrustNet (BVI) Limited  THA;VGB;IDN;SGP
    
    The Node with the most END edges has [1m37338[0m links and is :
               source country_codes
    type                           
    address  offshore           VGB


### Study nodes that are isolated


```python
node_ids = {x[0] for x in nodes.index.values}
linked_nodes = set(edges.START_ID.append(edges.END_ID).values.tolist())

isolated_nodes = node_ids.difference(linked_nodes)

print("There are", color(str(len(isolated_nodes))), "isolated nodes (0 edge connecting them)")
```

    There are [1m3277[0m isolated nodes (0 edge connecting them)



```python
mask_isolated = nodes.index.get_level_values(0).isin(isolated_nodes)
nodes[mask_isolated].describe()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>source</th>
      <th>name</th>
      <th>country_codes</th>
      <th>jurisdiction</th>
      <th>incorporation_date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>3277</td>
      <td>3261</td>
      <td>428</td>
      <td>184</td>
      <td>145</td>
    </tr>
    <tr>
      <th>unique</th>
      <td>4</td>
      <td>3233</td>
      <td>73</td>
      <td>3</td>
      <td>109</td>
    </tr>
    <tr>
      <th>top</th>
      <td>paradise</td>
      <td>THE BEARER</td>
      <td>IMN</td>
      <td>KNA</td>
      <td>1980-01-01 00:00:00</td>
    </tr>
    <tr>
      <th>freq</th>
      <td>3069</td>
      <td>21</td>
      <td>106</td>
      <td>149</td>
      <td>9</td>
    </tr>
    <tr>
      <th>first</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>1980-01-01 00:00:00</td>
    </tr>
    <tr>
      <th>last</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>2011-08-05 00:00:00</td>
    </tr>
  </tbody>
</table>
</div>



## Planning

We are planning to infer using the edges:
- all the jurisdictions some nodes such as *officer* have links to, in order to see if see if they are connected to accounts in different jurisdictions.
- for *entity* the country_codes, which would be the country of the people that are connected to this jurisdiction. 
