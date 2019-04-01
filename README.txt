Goal:  A concise illustration of the influence of latitude
on current weather conditions.

Rationale:  A demonstration of how "big data" in real time
can answer this somewhat mundane question in a richer and more 
detailed way than a heuristic model is an effective way of 
communicating the value of "big data" to an audience of non-experts.

Special notes:  Requirements

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
Kögl, however, because only the database from citipy is used, the kdtree 
source code is not required.  Rather, the notebook
expects a database file named worldcities.csv to be present in a 
folder named citipy.  The easiest way to make sure the file is 
where it should be is simply to install the citipy library.

In addition, the notebook requires the t-distribution function
from scipy.stats, so scipy.stats must be installed.  Other libraries
imported include datetime, time, and urllib.  An additional third-party
source code file, haversine.py, is also required in the same folder 
enclosing the notebook.

The demo runs in Jupyter notebook. To run the notebook
requires a working Internet connection to access real-time weather
data.  Although the data transferred is minimal, you may incur 
charges for transferring data.  Also please be aware that the demo
makes a large number of requests from the OpenWeatherMap API, so
please be considerate of the free service being provided.  

Helpful background info:
This demo will provide a "live" aggregation of the temperature, 
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
as described above.  Third-party libraries openweathermapy and citipy,
and the third-party haversine.py source file, are required.  Scipy
(specifically the scipy.stats module) is also required..  If all 
these file requirements are met, the user can simply run the notebook
without providing further input. 

Outputs:
Graphics will appear in the Jupyter notebook and be saved into .png
files in a folder called output_data.  A list of the cities used
and their weather conditions will also be placed in this folder in 
csv format.  The notebook will also generate a log of each weather
data request made, both in the notebook and in a file called
log.txt in the folder output_data.  The final cells in the notebook
contain tables of the statistical significance of correlations
between all of the measured weather variables and both latitude and
distance from the equator.  

General procedure:
S:  Generate a random sample of cities for collection of current 
weather data using citipy to link city names to coordiantes.
Q:  Use the OpenWeatherMap API to query the weather for these
cities and store the data.
P:  Use matplotlib to create and save the graphs
A:  Perform statistical analyses to identify significant correlations

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
spot is in the middle of the ocean, so the nearest city in Rikitea
in French Polynesia -- almost 3,000 miles away.  Although we can
use the coordinats of the weather station rather than the coordiantes
requested to build a valid correlation, we will be using an odd and
uncontrolled sampling scheme, which is not preferable.  

The "nearest city" function also has some flaws, namely: 1) it does 
not correctly handle the meridian at 180 deg E/W.  Nearer cities
on the opposite side of this meridian will not be found, 2) it uses
a "flat Earth" formula for distance that treats longitude and latitude
as "equally large" in space.  In reality, degrees of longitude are
smaller (by 30% at 45 degrees latitude and 50% at 60 degrees latitude)
so that the algorithm will tend to pick more distant cities north
and south of the target location as opposed to nearby ones east or
west.  This is undesirable for controlling latitude.

A further issue with random sampling of lat/long pairs is that it 
assumes a uniform distribution of latitude and longitude points on
Earth's surface.  In fact, because the Earth is a sphere, there are 
far more points at the equator than there are at a parallel near the 
pole.  Because there are many more points at the equator, if we
want a representative sample of city locations, we should sample 
random points on the Earth's surface rather than random lat / long
pairs, which will oversample polar regions.  

To correctly sample points on Earth's surface, we can sample the 
inverse cosine of latitude randomly, while sampling longitude randomly.  
Doing so generates points weighted by the cosine of latitude, which
is proportional to the surface area of Earth at that latitude.  To 
avoid odd forms of bias introduced by sampling cities far away from the 
target point, we will only add a city to the sample if it lies within
a relatively small distance (adjustable as a program parameter)
from the target point.  Consequently, we will generate many more points 
than cities, however, computationally, this causes a very low impact
on run time. (Most of the run time is spent waiting for API requests).

To correctly calculate distance, we will use the haversine distance 
function, which takes into account the contraction of longitude near 
the poles.  

Duplicates will be tracked on the basis of city name.  This will
eliminate some cities that are not duplicates but share a name (e.g.
if we sample Columbus, OH, we will not sample Columbus, GA), but the 
effect on sampling is expected to be negligible.  

Q:  To minimize load on the OpenWeatherMap server, we will run and log
queries in small batches (20, size adjustable). After each batch, we 
check the number of cities gathered and stop once we reach the desired 
target (also adjustable).  The program will also be set up with a max
number of batches to attempt, so that it will not keep querying forever.  
The program will have a parameter to limit the number of target 
coordinates generated per batch, to avoid getting stuck in an infinite 
loop while searching for cities.  These "safety" parameters will be
user adjustable.  After each successful query, we will parse 
temperature (in deg F), humidity (in %), cloud cover (in %), and 
wind speed (in mph), along with lat / long coordinates, city name
country, and a date/time stamp. The list of cities to be excluded and
the actual queries will be by city name only. There is the possibility,
if multiple query names return the same city, we could duplicate data,
but in testing this behavior has never been seen for the 
OpenWeatherMap API.  If this happens, the user will be warned and log
file will show fewer unique cities than queries.  

P:  Using the latitude as the x-axis, and the four weather data columns
as the y-axis, we will create four scatterplots and label them.  

A:  For each weather variable, we can test the hypothesis that the 
varuabke us independent of both absolute latitude as well as distance 
from the equator (absolute value of latitude times 111.13 km per degree).
Because we cannot validate the assumptions of linearity
homoskedacity, and absence of outliers needed for linear correlations,
we will use Spearman's correlation coefficient to check for effects,
rather than Pearson's correlation coefficient.  The use of the Spearman 
coefficient will cost us a quantitatively validated threshold for 95% 
confidence, so we will describe confidence qualitatively.  The approach
recommended is the use of the significance cut-offs for 95% confidence
with the Pearson correlation, translated into qualitative terms, and 
to treat correlations that are close to 95% significance as marginal
rather than definite.  To handle this situation, the correlations will 
be labeled "marginal" if the equivalent Pearson correlation coefficient 
would have been signfiicant at 95% to 99% confidence, and "likely" if
the equivalent Pearson coefficient would have been significant at 99%.  

