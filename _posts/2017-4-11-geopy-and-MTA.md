---
layout: post
title: Using GeoPy to Filter MTA Subway Stations
---

For our first project at Metis we analyzed [MTA turnstile data](http://web.mta.info/developers/turnstile.html). Our task was to recommend stations for a fictitious non-profit to place canvassers to collect email addresses for an upcoming event. My group decided to recommend stations that were among the busiest in the city and in proximity to the largest tech companies and a handful of upcoming events.  

I have experience using [Geographic Information Systems (GIS)](https://en.wikipedia.org/wiki/Geographic_information_system) software so my mind went immediately to coordinate pairs. I volunteered to filter the stations for proximity and map the results after my group members compiled the necessary data. To do this I decided to use the [Google Maps API](https://developers.google.com/maps/) to collect coordinates for all of the stations, companies, and event venues of interest. I then used the [geopy](https://pypi.python.org/pypi/geopy) Python library to calculate the distance from each station to each company and event. As a group we decided a tenth of a mile was a reasonable distance to filter by, so the result was a list of all stations within a tenth of a mile of either a company or an event. I then wrote the coordinates of the stations, companies, and events to a csv file to import into [QGIS](http://qgis.org/en/site/index.html), an open-source GIS software.  

Once I registered for a Google Maps API key, I started to play around with the API using the Python [`requests`](http://docs.python-requests.org/en/master/) library. The result was a function `get_loc` that constructed a URL request from strings and extracted the latitude and longitude coordinates from the API query. 

```python
import requests

def get_loc(places,req1,req2):

    locs = {}
    counter = 0
    
    for place in places:
        r = requests.get(req1 + place + req2)
        
        try:
            results = r.json()['results'][0][u'geometry'][u'location']
            lat = results[u'lat']
            lng = results[u'lng']
            locs[place] = [lat,lng]
            
        except:
            pass
            
    return locs
```  
  
`get_loc` takes in a list of locations to search for with the API and a string for each half of the URL request. The locations should be strings formatted to work well with URLs. This required swapping white-space characters for `+`, for instance. The function constructs the full request by placing the location in between each half. Here's an example of a full request that queries for the [Fulton St station](https://www.google.com/maps/place/Fulton+St,+New+York,+NY/@40.7102288,-74.0099294,17z/data=!3m1!4b1!4m5!3m4!1s0x89c25a17fed80351:0xf3596b913f0c9185!8m2!3d40.7102288!4d-74.0077407).


`https://maps.googleapis.com/maps/api/geocode/json?address=FULTON+ST+mta+subway+NY+NY&key=API_KEY`

The output of the Google Maps API is a .json file. A portion of the output is below. You can see the full output [here](fulton_st.json).  

```json
{
   "results" : [
      {
         "formatted_address" : "Fulton St Subway Station, New York, NY 10038, USA",
         "geometry" : {
            "location" : {
               "lat" : 40.709373,
               "lng" : -74.00832579999999
         },
         "place_id" : "ChIJZRVRExhawokR_OhyrnV9EtM",
         "types" : [
            "establishment",
            "point_of_interest",
            "subway_station",
            "transit_station"
         ]
      }
   ],
}
```

The .json file confirms that we are indeed looking at the Fulton St subway station. `get_loc` digs through the .json structure to access the `"lat"` (latitude) and `"lng"` (longitude) entries. It ignores any requests that fail. The requests rarely failed so I manually entered the coordinates for any that did. After gathering all of the coordinates for the subway stations I used `get_loc` to do the same for the companies and events.

Once I had all of the necessary coordinate pairs I was ready to start measuring distances. I came across the `geopy` library, which makes calculating distances between coordinate pairs very easy. I knew I was going to need to calculate the distance between many coordinate pairs so I wrote a function `calculate_distance` to automate the calculation.

```python
from geopy.distance import vincenty
from geopy.point import Point

def calculate_distance(a,b):

    pointA = Point(a)
    pointB = Point(b)
    
    return vincenty(pointA, pointB).miles
```

`calculate_distance` simply takes two coordinate pairs and calculates the distance between them using `geopy`'s `vincenty` function. `calculate_distance` first converts each coordinate pair into a `geopy` point object using the `Point` function. It then calls the `vincenty` function on the two points. The `vincenty` function uses the [Vincenty method](https://en.wikipedia.org/wiki/Vincenty's_formulae) of calculating the distance between two points by approxmating the earth as a spheroid, [which it is](https://en.wikipedia.org/wiki/Figure_of_the_Earth). This method is more accurate than modeling the earth as a simple sphere. The `vincenty` function allows you to request the unit of distance with a simple appendage to the function call, which is remarkably convenient.

The next step was to write a function, `in_proximity`, to use `calculate_distance` to calculate the distance between several coordinate pairs. 

```python
def in_proximity(location,coords,radius):
   
    proximates = {}
    
    for entry in coords.keys():
        coord = coords[entry]
        distance = calculate_distance(coord,location)
        
        if distance < float(radius):
            proximates[entry] = "%.3f" % distance
            
    return proximates
```

`in_proximity` takes a single coordinate pair, `location`, a list of coordinate pairs, `coords`, and a distance, `radius`, in miles. The function cycles through each set of coordinates in `coords` and calculates the distance between those coordinates and `location`. If the distance is less than `radius`, the function stores the coordinates. `in_proximity` returns the set of coordinates within `coords` that fall within the specified distance from `location`.

The last step was to scale `in_proximity` up to take a list of locations. Instead of altering `in_proximity` I wrote a new function `collect_proximates` that calls `in_proximity` on a list of locations.

```python
def collect_proximates(locations,coordinates,distance=0.1):
    
    proximate_collection ={}
    
    for name, coord in locations.items():
        proximate_collection[name] = in_proximity(coord,coordinates,distance)
        
    return proximate_collection
```

`collect_proximates` returns a dictionary of the locations with each set of coordinates in `coordinates` that is within the specified distance of the location.

In the context of the project, I passed `collect_projects` a list of the companies and event locations and the coordinates of every subway station. The result was the set of subway stations within a tenth of a mile of either an event or a company of interest.

Here's a sample of the output.

```python
'Aol': {'8+ST+NYU': '0.060',
        'ASTOR+PL': '0.050'},
'Bloomberg': {'GRD+CNTRL+42+ST': '0.081'},
'Facebook': {'8+ST+NYU': '0.055'},
'Google': {'8+AV': '0.100'},
'Microsoft': {'42+ST+PORT+AUTH': '0.069'},
'Oscar': {"B'WAY+LAFAYETTE": '0.053'}
```

From the output of `collect_projects` I was able to extract the list of stations that were within reasonable proximity to either an event venue or company of interest. I cross-referenced this list with the list of the busiest stations to find the intersecting set of stations. Our recommendation consisted of the list of these stations. 

The final step was to map our results. I imported the coordinates of the events, companies, and recommended stations into QGIS using csv files. I was then able to plot the coordinates as points on a map of NYC.

![MTA Map]({{ site.baseurl }}/images/mta_map.png "MTA Map")

The results are not suprising if you are at all familiar with the NYC subway. It's always good to check if your results are reasonable, though. 