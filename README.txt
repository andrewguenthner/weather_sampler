Goal:  A concise illustration of the influence of latitude
on current weather conditions.

Rationale:  A demonstration of how "big data" in real time
can answer this somewhat mundane question in a richer and more 
detailed way than a heuristic model is an effective way of 
communicating the value of "big data" to an audience of non-experts.

Special notes: 

***API Key Required***
Running this demonstration requires the user to first obtain
an API key from OpenWeatherMap.  Documentation and instructions
for how to do so are available at the following url:
https://openweathermap.org/api

Once you have obtained the API key, you will need to copy it into
the file "config.py" contained in the directory where the notebook
containing the demonstration is stored.  This file should have
a single line of text which reads as follows:
api_key = "{YOUR API KEY HERE}"
Simply open the file with a text editor and replace the {} and 
everything in between with the API key, which you will receive
via email from OpenWeatherMap once you follow the sign-up 
instructions. Be aware that it may take some time for the API
key to become active once you have obtained it.  

Note that the OpenWeatherMap's free service is subject to use and
availability limitations.  The demo contains features that reduce
the load on the OpenWeatherMap servers, please do not disable these
features.

***Third-party Python Libraries Required***
This demonstraiton makes use of the third-party Python libraries
openweathermapy (copyright: (c) 2015 by Stefan Kuethe, released
under the GNU GPLv3 license) and citiPy 0.0.5 available on
PyPI at https://pypi.python.org/pypi/citipy. These libraries
are included as sub-folders in the folder containing the notebook 
demo for convenience.  Citipy makes use of kdtree.py, a library for 
creating and using trees with distance, version 0.16 by Stefan
Kögl, which is also included.

The demo runs in Jupyter notebook, and requires the files config.py
as well as the libraries to run successfully.  The notebook also
requires a working Internet connection to access real-time weather
data.  Although the data transferred is minimal, you may incur 
charges for transferring data.  Also please be aware that the demo
makes a large number of requests from the OpenWeatherMap API, so
please be considerate of the free service being provided.  

Issues to consider:
This demo will provide a "live" demonstration of the temperature, 
cloud cover, humidity, and wind speed in several hundred cities
around the world, graphed by latitude.  It shows how the weather
near cities changes as latitude changes.  Note that "weather near
cities" is not the same as "the weather" in general.  The world's
cities are entirely on land for the foreseeable future, and they
tend to be located near large bodies of water, away from extremely
high elevations, away from deserts, and away from the poles in 
general.  The world's cities are also not evenly distributed by
longitude, so at certain times a variable portion of the world's urban
areas will be in daylight or in summer vs. winter.  These factors
will influence the distribution of temperature by latitude, so that
the results for a given moment may be systematically different 
from those at a different time of day and year.  

Inputs:
The notebook requires the user's API key in the file config.py
as described above.  Third-party libraries openweathermapy, citipy,
and kdtree are required.  Citipy also comes with a list of cities
stored in a file named worldcities.csv.  

Outputs:
Graphics will appear in the Jupyter notebook and be saved into .png
files in a folder called output_data.  A list of the cities used
and their weather conditions will also be placed in this folder in 
csv format.  The notebook will also generate a log of each weather
data request made, both in the notebook and in a file called
log.txt in the folder output_data.

General procedure:
S:  Generate a random sample of cities for collection of current 
weather data using Citipy to link city names to coordiantes.
Q:  Use the OpenWeatherMap API to query the weather for these
cities and store the data.
A:  Use the openweathermapy module to prepare the data for analysis
P:  Use matplotlib to create and save the graphs

Note that the demo will generate a very verbose log of connection
information.  This is meant to help you be aware of the external
data transfers that are taking place and diagnose any issues.

Detailed procedure and design notes:
S1.  Sampling scheme:
Because we are interested in variables as a function of latitude we
need to *control* for latitude. Although we could randomly sample
latitude and longitude pairs and use citipy's "nearest city" function, 
that function does not limit the distance to the nearest point returned. 
So, for instance, if we request lat(itude) = -60 (60 deg S), 
long(itude) = -115 (115 deg W), we actually get a city at
lat = -23, long = -135, far different than what we asked for.  The
reason is that, like most random combinations of lat/long, this
spot is in the middle of the ocean, far from land, and we get Rikitea
in French Polynesia -- almost 3,000 miles away.  That's a sure
way to come to false conclusions, particularly about the Southern
Hemisphere where ocean is especially predominant.  

The "nearest city" function also has some flaws, namely: 1) it does 
not correctly handle the meridian at 180 deg E/W.  Nearer cities
on the opposite side of this meridian will not be found, 2) it uses
a "flat Earth" formula for distance that treats longitude and latitude
as "equally large" in space.  In reality, degrees of longitude are
smaller (by 30% at 45 degrees latitude and 50% at 60 degrees latitude)
so that the algorithm will tend to pick more distant cities north
and south of the target location as opposed to nearby ones east or
west.  This is especially bad for controlling latitude.  

A further issue with random sampling is that it assumes a uniform
distribution of latitude and longitude points on Earth's surface.
In fact, because the Earth is a sphere, there are far more points
at the equator than there are at a parallel near the pole (e.g. 
85 deg N).  Because there are many more points at the equator, we
should expect that there may well be a wider variety of weather.
We can capture this phenomenon by putting more points in scatterplots
at locations near the equator, and fewer near the poles.  As it 
happens, there are virtually no cities near the poles, so our
representation of the weather in cities should be sparse.  

To both control for latitude and sample correctly, we can still
use a random distribution, but we will break the distribution
into bins and adjust the potential sample size in each bin to reflect 
the relative area of that particular latitude band, which is
porportional to the cosine of the latitude.  Within a bin, we will
pick a location at random, then look for the closest city within
a distance equal to the width (in latitude) of the bin.  For distance,
we will use available Python code for the Haversine function, which
correctly computes distances between lat/long points.  If no cities
are found within that radius, no cities will be appended to the list.
If more than one city is found, the closest city will be appended,
provided it has not already been selected.

If we make the number of attempts proportional to the cosine of latitude,
then we will ssample about the same fraction of the surface of the planet
uniformly.  Fewer successful attempts will mean fewer populated areas,
thus we will automatically generate cities in approximate proprotion
to populated landmass.  We can do this fairly simply by using a biased
random distribution for latitude and a random distribution for longitude.

Because we would like to have a sample of over 500 cities with weather
data, and because not every city will have such data, we will iteratively
sample until we reach 1000 candidate cities.  We will use Citipy to 
retrieve city names and coordinates but use the Haversine distance 
calculation.  The cities will be stored in a list for querying.

Q:  To minimize load on the OpenWeatherMap server, we will run and log
each query one at a time (on screen and in file), until we reach 500
data points.  If we fail to reach 500,  we will repeat S1 until we 
are successful.  We will build a DataFrame frame to retain only the cities
with successful queries.  For each query, we will parse temperature (in F),
humidity (in %), cloud cover (in %), and wind speed (in mph) and store
them, along with lat / long coordinates, country, and a date/time stamp.

A:  Using the latitude as the x-axis, and the four weather data columns
as the y-axis, we will create four scatterplots and label them.  For
each, we can also test the hypothesis that the variable is independent
of latitude.  Because we cannot validate the assumptions of linearity
homoskedacity, and absence of outliers needed for linear correlations,
we will use Spearman's correlation (with latitude N to S and absolute
value of latitude as independent variables) to check for effects. 

