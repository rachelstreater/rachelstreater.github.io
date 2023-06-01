---
layout: post
title:  code snippet - using an API
date:   2023-06-01
description: using a geolocation API in pandas to convert coordinates
tags: images
categories: python, geospatial
---

Code snippet to replace data in a pandas dataframe with the response from an API call.

### The data:

We have a dataframe with location in X/Y format, which we want to convert to Latitude/Longitude for plotting and analysis. 

| location_index   |   latitude |   longitude |   easting |   northing |\n
|:-----------------|-----------:|------------:|----------:|-----------:|\n
| 197901A1BAW34    |            |             |    198460 |     894000 |\n
| 197901A1BFD77    |            |             |    406380 |     307000 |\n
| 197901A1BGC20    |            |             |    281680 |     440000 |\n
| 197901A1BGF95    |            |             |    153960 |     795000 |\n
| 197901A1CBC96    |            |             |    300370 |     146000 |


### Imports: 

````markdown
```python
import pandas as pd
import numpy as np
import requests
import logging
import pyproj
```
````

### Create a function to query the API: 
Need to define the api url and parameters.

````markdown
```python
def bng_to_lat_lon(params):
    try:
        api_url = f'http://webapps.bgs.ac.uk/data/webservices/CoordConvert_LL_BNG.cfc?method=BNGtoLatLng'
        response = requests.get(api_url, params=params)
        return [response.json()['LATITUDE'], response.json()['LONGITUDE']]
    except requests.exceptions.RequestException as e:
        raise SystemExit(e)
```
````

### Build the request parameters by creating a dictionary from the pandas dataframe:

````markdown
```python
request = df[['easting', 'northing']].set_index('location_index').to_dict(orient='index')
```
````

### Update the new coordinates:

Make sure to name the parameters as the API expects them, in this case *easting* and *northing*.

````markdown
```python
for i in request.index:
    params = dict(accident.loc[i][['easting', 'northing']])
    lat, lon = convert_bng_to_lat_lon(params)
    accident['latitude'][i] = lat
    accident['longitude'][i] = lon
```
````


### The result: 

| accident_index   |   latitude |   longitude |   easting |   northing |\n
|:-----------------|-----------:|------------:|----------:|-----------:|\n
| 197901A1BAW34    |    57.8898 |    -5.40163 |    198460 |     894000 |\n
| 197901A1BFD77    |    52.6607 |    -1.90711 |    406380 |     307000 |\n
| 197901A1BGC20    |    53.8427 |    -3.79979 |    281680 |     440000 |\n
| 197901A1BGF95    |    56.9805 |    -6.05094 |    153960 |     795000 |\n
| 197901A1CBC96    |    51.2045 |    -3.42749 |    300370 |     146000 |


### A better way for this particular task:

In benchmark testing, the previous implementation processed 512 rows per minute, whereas the following method was able to process 40,814 rows per minute. The bottleneck was the API calls which were running at around 8 per second.


````markdown
```python
from pyproj import Transformer
transformer = Transformer.from_crs("EPSG:27700", "EPSG:4326")

for i in request.index:
    lat_lon = transformer.transform(accident.loc[i]['easting'], accident.loc[i]['northing'])
    accident['latitude'][i] = lat_lon[0]
    accident['longitude'][i] = lat_lon[1]
```
````