---
layout: post
title:  Code Snippet - Using an API with Pandas
date:   2023-06-01
description: Geolocation API in Pandas to convert coordinates with comparison to pyproj package
tags: python, geospatial
#categories: python, geospatial
---


We have a dataframe with location in X/Y format, which we want to convert to Latitude/Longitude and put back into the dataframe for plotting and analysis.

Create a function to query the API. Need to define the API url and parameters.

```python
import pandas as pd
import numpy as np
import requests

def bng_to_lat_lon(params):
    try:
        api_url = f'http://webapps.bgs.ac.uk/data/webservices/CoordConvert_LL_BNG.cfc?method=BNGtoLatLng'
        response = requests.get(api_url, params=params)
        return [response.json()['LATITUDE'], response.json()['LONGITUDE']]
    except requests.exceptions.RequestException as e:
        raise SystemExit(e)
```

Build the request parameters by creating a dictionary from the pandas dataframe:


````python
request = df[['easting', 'northing']].set_index('location_index').to_dict(orient='index')
````

Update the new coordinates:


Make sure to name the parameters as the API expects them, in this case *easting* and *northing*.


````python
for i in request.index:
    params = dict(accident.loc[i][['easting', 'northing']])
    lat, lon = convert_bng_to_lat_lon(params)
    accident['latitude'][i] = lat
    accident['longitude'][i] = lon
````


Run the code and the update is complete.


### A more efficient solution:

In benchmark testing, the previous implementation processed 512 rows per minute, whereas the following method was able to process 40,814 rows per minute. The bottleneck was the API calls which were running at around 8 per second.



```python
from pyproj import Transformer
transformer = Transformer.from_crs("EPSG:27700", "EPSG:4326")

for i in request.index:
    lat_lon = transformer.transform(accident.loc[i]['easting'], accident.loc[i]['northing'])
    accident['latitude'][i] = lat_lon[0]
    accident['longitude'][i] = lat_lon[1]
```
