## **Automate OSINT workflows with GIS**

## **Overview**

Open-source intelligence (OSINT) is the collection and analysis of open-source (OS) datasets to provide intelligence products. The Armed Conflict Location & Event Data Project (ACLED)[^1]  collects real-time data on the locations, dates, actors, fatalities of violence, and protests around the world (ACLED, 2022). As each event has an associated location and date, GIS can be employed to spatially, and temporally analyse this dataset to extract and visualise trends for intelligence products.

The ACLED Analysis tool was developed to automate such a workflow using the publicly available ACLED dataset; the deployed example will focus on the current conflict within Ukraine. It will detail the number of events and fatalities over time, determine the most affected regions, and spatially plot the temporal patterns of the conflict; five charts and five maps are produced to display these derived data.


[^1]: The Armed Conflict Location & Event Data Project (ACLED); [http://acleddata.com](http://acleddata.com)

## **Setup and Installation**

The code for this GIS tool is publicly hosted on GitHub and can be accessed from the EGM722-ACLED-Analysis repository; [https://github.com/BW-GIS-Developer/EGM722-ACLED-Analysis.git](https://github.com/BW-GIS-Developer/EGM722-ACLED-Analysis.git).

The code was developed using Jupyter Notebooks and Visual Studio Code and should be run using an integrated development environment (IDE) that can support the .ipynb file type; It is recommended that Anaconda Navigator[^2]  is used for this.

Several python dependencies are needed for the tool to run, these are detailed in table 1. These dependencies should be installed into a virtual environment by using the [environment.yml](https://github.com/BW-GIS-Developer/EGM722-ACLED-Analysis/blob/main/environments.yaml) file saved in the GitHub repository; this requires Anaconda to be installed as it uses conda-forge to install and saves the virtual environment within the Anacoda3 directory. Alternatively, they can be installed individually using the conda-forge channel from within the conda command prompt.

*Table 1: Python dependencies*

| Dependency | Version | Manual install through conda           |
|:-----------|:-------:|---------------------------------------:|
| python     |  3.8.8  | conda install -c anaconda python       |
| geopandas  |  0.9.0  | conda install -c conda-forge geopandas |
| cartopy    |  0.18.0 | conda install -c conda-forge cartopy   |
| notebook   |  6.2.0  | conda install -c onda-forge notebook   |
| h3-py      |  3.7.3  | conda install -c conda-forge h3-py     |
| jenkspy    |  0.2.0  | conda install -c conda-forge jenkspy   |

The datasets required to run the code are detailed in table 2. To use the ACLED data export tool an account is required, this is free and limits the user to a small number of annual downloads. These datasets can also be found in the GitHub repository as a [zip file](https://github.com/BW-GIS-Developer/EGM722-ACLED-Analysis/blob/main/Python%20Code/Data.zip).

*Table 2: Required datasets*

| Dataset                    | File type | source                                                                                                                                   |
|:---------------------------|:---------:|-----------------------------------------------------------------------------------------------------------------------------------------:|
| ACLED[^3]                  | csv       | [https://acleddata.com/data-export-tool/](https://acleddata.com/data-export-tool/)                                                       |
| Country Boundary (Global)  | shapefile | [https://hub.arcgis.com/datasets/a21fdb46d23e4ef896f31475217cbb08_1](https://hub.arcgis.com/datasets/a21fdb46d23e4ef896f31475217cbb08_1) |
| Country Boundary (Ukraine) | shapefile | [https://hub.arcgis.com/datasets/a21fdb46d23e4ef896f31475217cbb08_1](https://hub.arcgis.com/datasets/a21fdb46d23e4ef896f31475217cbb08_1) |
| Ukraine Oblasts            | shapefile | [https://hub.arcgis.com/datasets/dd3afd55a8fc428daef8ae395e8cd582_0](https://hub.arcgis.com/datasets/dd3afd55a8fc428daef8ae395e8cd582_0) |

Commonly accessed global variables are defined the at start of the code. These can be changed, alongside other variables which will be detailed later, to allow the tool to work with other downloads of ACLED data. Those which may require change are detailed below:

```python
# ACLED dataset - read csv
ukraine_acled_pandas = pd.read_csv(r"Data\ACLED\UkraineACLED.csv")

# Ukraine country boundary
ukraine_boundary_geopandas = gpd.read_file(r"Data\Boundaries\UKR_Boundary.shp")

# Ukraine oblast bondaries
ukraine_oblast_geopandas = gpd.read_file(r"Data\Boundaries\UKR_Adm.shp")

# Start, middle and end dates
date_start, date_mid, date_end = ((datetime(2022,1,1), datetime(2022,2,24), datetime(2022,4,15)))

# Colours used for map products
colour_ramp = ["ivory", "lightcoral", "firebrick", "darkred"]

# Buffer values used for map products
buffer_values = [5000, 10000, 15000, 20000, 25000, 30000]
```

[^2]: Anaconda Navigator; [https://docs.anaconda.com/anaconda/install/](https://docs.anaconda.com/anaconda/install/)
[^3]: ACLED export filters applied - From: 01 Jan 22, To: 15 Apr 22, Region: Europe, Country: Ukraine

## **Methodology**

The tool was developed to provide a solution to automate a simulated OSINT workflow, using pre-existing publicly available datasets; in this case to display trends within a subset of ACLED data. A total of 11 functions were developed to perform various data engineering and analytical processes, allowing the code the flow and follow a logical workflow; the outline of which is detailed below:

    1.	Import required python modules
    2.	Define commonly accessed global variables
    3.	Create buffers
    4.	Create 3 subsets of the input ACLED data
        a.	Entire date range (start date to end date)
        b.	First date section (start date to middle date)
        c.	Last date section (middle date to end date)
    5.	Calculate statistics within the entire date range subset
        a.	Daily counts for events
        b.	Daily counts for fatalities
        c.	Accumulative counts for events
        d.	Accumulative counts for fatalities
        e.	Daily counts for events split by responsible actor
    6.	Calculate statistics by oblast regions within the entire date range subset
        a.	Total event counts
        b.	Total fatality counts
    7.	Aggregate events into hexbins
        a.	Total events
        b.	Total fatalities
    8.	Create a series of graphs and maps to display these derived datasets
    9.	Run all functions


### **Import required python modules**

All python modules required for the script to execute without errors are imported, some of these are available with the native installation of python 3.x, and the rest are required to be installed separately; refer to the setup and installation for these dependencies. 

```python
import geopandas as gpd
import pandas as pd
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
import cartopy.crs as ccrs
import matplotlib.patches as mpatch
import matplotlib.patheffects as pe

from cartopy.feature import ShapelyFeature
from datetime import datetime
from shapely.geometry import Polygon, Point

import h3, jenkspy
```

### **Define commonly accessed global variables**

Several variables are accessed multiple times throughout the script, these are defined at the global level to access within functions and save repeating lines of code. These variables relate to input datasets, boundary extents, map coordinate systems, date ranges, and colour schemes. If alternative ACLED datasets are used, some of these must be altered for the script to execute correctly; refer to the setup and installation for these variables and changes.

```python
# ACLED dataset - read csv
ukraine_acled_pandas = pd.read_csv(r"Data\ACLED\UkraineACLED.csv")

# ACLED dataset - create list of XY coordinates as a Point
acled_geometry = [Point(xy) for xy in zip(ukraine_acled_pandas.longitude, ukraine_acled_pandas.latitude)]

# ACLED dataset - read dataset as a geodataframe by setting the geometry
ukraine_acled_geopandas = gpd.GeoDataFrame(ukraine_acled_pandas, crs="EPSG:4326", geometry=acled_geometry)

# Ukraine country boundary
ukraine_boundary_geopandas = gpd.read_file(r"Data\Boundaries\UKR_Boundary.shp")

# Ukraine oblast bondaries
ukraine_oblast_geopandas = gpd.read_file(r"Data\Boundaries\UKR_Adm.shp")

# Global country boudnaries
global_boundary_geopandas = gpd.read_file(r"Data\Boundaries\GLB_Bnd.shp")

# Calculate boundary extent for map display
xmin, ymin, xmax, ymax = ukraine_boundary_geopandas.total_bounds

# Set map coordinate system
map_crs = ccrs.Mercator()

# Start, middle and end dates
date_start, date_mid, date_end = ((datetime(2022,1,1), datetime(2022,2,24), datetime(2022,4,15)))

# ACLED dataset columns to keep (date will be calculated later on)
columns_acled = ["actor1", "event_date", "date", "fatalities", "geometry"] 

# Colours used for map products
colour_ramp = ["ivory", "lightcoral", "firebrick", "darkred"]

# Buffer values used for map products
buffer_values = [5000, 10000, 15000, 20000, 25000, 30000]

# Transparency values used for map products
transparency_values = [0.05, 0.1, 0.15, 0.2, 0.25, 0.3]

# Shapely Feature for the Global boundaries
GLB_outline = ShapelyFeature(global_boundary_geopandas["geometry"], map_crs, edgecolor='k', facecolor='darkseagreen')

# Shapely Feature for the Ukrainian oblasts
oblast_outline = ShapelyFeature(ukraine_oblast_geopandas["geometry"], map_crs, edgecolor='k', facecolor='tan')
```

### **Create buffers**

A function called create_buffers is used to buffer the country of interest multiple times based on the global variable buffer_values. In doing so, the buffer can be applied to the map products as a feathered effect, allowing the country of interest to stand out.

### **Create 3 subsets of the input ACLED data**

This section of code relies upon two functions:

    1.	convert_to_datetime
    2.	calculate_datetime_ranges

The first uses the strptime() function available in the datetime module to convert the original ACLED text date, formatted as 15-Apr-22 into a datetime date.

```python
def convert_to_datetime(date):
    
    return datetime.strptime(date, "%d %B %Y")
```

The second creates 3 pandas dataframes for each of the date ranges defined. The convert_to_datetime function is applied to each of the dates in the original CSV, writing the output into a new column called date. This column is then used to filter out the dataframe rows where the date falls within each range, saving as a new dataframe, and sorting by date.

```python
def calculate_datetime_ranges(acled, columns, start, mid, end):

    # Create a new column called "date" and convert the original string dates into datetime
    acled["date"] = acled["event_date"].apply(convert_to_datetime)

    # Create geopandas dataframes for date range start - end
    acled_daterange_se = acled[(acled["date"] >= start) & (acled["date"] <= end)]

    # Create geopandas dataframes for date range start - mid
    acled_daterange_sm = acled[(acled["date"] < mid) & (acled["date"] >= start)]
    
    # Create geopandas dataframes for date range mid - end
    acled_daterange_me = acled[(acled["date"] >= mid) & (acled["date"] <= end)]
    
    return acled_daterange_se[columns].sort_values(by=["date"]), acled_daterange_sm[columns].sort_values(by=["date"]), acled_daterange_me[columns].sort_values(by=["date"])
```

### **Calculate statistics within the entire date range subset**

This section of code relies upon one function: 

    1.	calculate_statistics

This function creates a list of unique dates within the ACLED subset spanning the entire date range, sorting the list by date. For each date, multiple calculations are made, stored as variables, and then stored into a python dictionary, which is later outputted as a pandas dataframe; these variables are:

    •	daily_count; The number of events
    •	daily_fatal; The number of fatalities
    •	daily_russian; The number of events where Russia is the actor
    •	daily_ukraine; The number of events where Ukraine is the actor
    •	daily_other; The number of events where another actor is responsible

Counter variables are used to calculate the accumulative sums for both events, and fatalities individually; respectively called count_sum and count_fatal.

If an alternative ACLED dataset is used instead of the deployed one, changes to the following sections will need to be made to work with the theme of choice:

    •	daily_russian: Update variable name and value within str.contains(“Russia”)
    •	daily_ukraine: Update variable name and value within str.contains(“Ukraine”)
    •	daily_other: Update variables used within the sum calculations
    •	statistics_by_date: Update dictionary keys for “russian_actor” and “ukraine_actor” 
        o	The append daily statistics section should also be updated to reflect this change 
        o	e.g., statistics_by_date["russian_actor"].append(daily_russian)

```python
def calculate_statistics(acled_all):
 
    # Store unique dates in a list
    unique_dates_set = set(acled_all["date"].tolist())
    unique_dates = list(unique_dates_set)
    unique_dates.sort()

    # Create dictionary to store statistics
    statistics_by_date = {
        "date" : [],
        "count" : [],
        "fatalities" : [],
        "count_sum" : [],
        "fatalities_sum" : [], 
        "russian_actor" : [],
        "ukraine_actor" : [],
        "other_actor" : []
    }
    
    # Create counter variables
    count_sum = 0
    count_fatal = 0

    # Iterate over unique dates to create daily statistics
    for date in unique_dates:

        # Calculate daily statistics
        daily_count = acled_all["date"].value_counts()[date]
        daily_fatal = acled_all.loc[acled_all["date"] == date, "fatalities"].sum()
        daily_events = acled_all[acled_all["date"] == date]
        daily_russian = len(daily_events[daily_events["actor1"].str.contains("Russia")])
        daily_ukraine = len(daily_events[daily_events["actor1"].str.contains("Ukraine")])
        daily_other = daily_count - daily_russian - daily_ukraine

        # Append daily statistics
        statistics_by_date["date"].append(date)
        statistics_by_date["count"].append(daily_count)
        statistics_by_date["fatalities"].append(daily_fatal)
        statistics_by_date["count_sum"].append(count_sum + daily_count)
        statistics_by_date["fatalities_sum"].append(count_fatal + daily_fatal)
        statistics_by_date["russian_actor"].append(daily_russian)
        statistics_by_date["ukraine_actor"].append(daily_ukraine)
        statistics_by_date["other_actor"].append(daily_other)

        # Update counters
        count_sum += daily_count
        count_fatal += daily_fatal

    return pd.DataFrame(statistics_by_date).sort_values(by=["date"])
```

### **Calculate statistics by oblast regions within the entire date range subset**

This section of code relies upon one function:

    1.	count_by_oblast

This function spatially joins events within the ACLED dataset to oblasts, where the event is contained within the oblast.

For each oblast, values for the number of events, and fatalities are calculated, stored as variables, and then stored in a python dictionary; these variables are:
	
    •	count; The number of events in the current oblast
    •	fatal; The number of fatalities in the current oblast

Once complete, this dictionary is converted into a pandas dataframe, enabling it to be merged with the global variable oblasts. The merge is based on the oblast region name, relating the geometry for each oblast as oblasts is a geopandas dataframe.

Finally, for each region, a symbology value is created and used later to apply the correct colour theme based on the number of events or fatalities in each oblast. To achieve this, the python module jenkspy is used to create values based on natural breaks (Jenks). Four classes are calculated based on natural groupings within the data.

If an alternative regional boundary dataset is used instead of the oblasts, changes to the following sections will need to be made to work with the theme of choice:

    •	The column name Region is used to access oblast names, this should be updated to reflect the new dataset

```python
def count_by_oblast(oblasts, acled):
    
    # Create dictionary to store statistics
    statistics_by_oblast = {
        "Region" : [],
        "count" : [],
        "fatalities" : []
    }

    # Perform Spatial Join
    acled_oblast_sj = gpd.sjoin(oblasts, acled, how='inner', lsuffix='left', rsuffix='right')

    # Get a list of all region names
    unique_regions = set(acled_oblast_sj['Region'].tolist())
    unique_regions = list(unique_regions)
    unique_regions.sort()

    # Update dictionary for regions, counts and fatalities
    for region in unique_regions:
        fatal = acled_oblast_sj.loc[acled_oblast_sj["Region"] == region, "fatalities"].sum()
        count = acled_oblast_sj["Region"].value_counts()[region]

        statistics_by_oblast["Region"].append(region)
        statistics_by_oblast["count"].append(count)
        statistics_by_oblast["fatalities"].append(fatal)

    # Convert dictionary into pandas dataframe
    statistics_by_oblast_pandas = pd.DataFrame(statistics_by_oblast)

    # Merge regions statistics to oblast regions to link geometry
    oblast_statistics = statistics_by_oblast_pandas.merge(oblasts[["Region", "geometry"]])

    # Calculate values for Natural Breaks Jenks
    oblast_statistics["NBJ"] = pd.cut(oblast_statistics["count"], bins=jenkspy.jenks_breaks(oblast_statistics["count"], nb_class=4), labels=[1,2,3,4], include_lowest=True)
    
    return oblast_statistics
```

### **Aggregate events into hexbins**

This section of code relies upon one function:

    1.	create_hexbins

This function aggregates events into hexbins using the Uber H3 hexagonal hierarchical geospatial indexing system[^4]. Although created now, the function is called later on by the create_hexbin_maps function, which allows associated maps and themes (events/fatalities) to be spatially plotted and dynamically symbolised.

For each event, the geometry is read to collate the XY coordinates. This X and Y are passed into the Uber H3 function geo_to_h3, which returns the unique id (UID) for the H3 hexagon the event is contained within. Each UID is stored as a key within a python dictionary, with the following values being assigned to the respective UID key:

    •	count: The number of events associated with the hexagon UID
    •	fatalities: The number of fatalities associated with the hexagon UID
    •	Boundary: The geometry of the Uber H3 hexagon
        o	To calculate the boundary, pass the hexagon UID into the Uber H3 function h3_to_geo_boundary

This python dictionary is then converted into a geopandas dataframe, using the hexagon vertices as the geometry. 

For each hexagon, a symbology value is created and used later to apply the correct colour theme based on the number of events or fatalities in each hexagon; this uses the same jenkspy method used previously for oblast regions.

Finally, when later called upon as a function, based on the associated map and theme a shapely feature is created, symbology applied, and added to the map.

[^4]: Uber H3 Hexagonal hierarchical geospatial indexing system [h3geo](https://h3geo.org/)

```python
def create_hexbins(acled, map, theme, sorted_NBJ, colour_ramp):

    # Create dictionaries for storing hexbin data
    acled_to_h3 = {}

    h3_statistics = {
        "Hexbin"  : [],
        "count" : [],
        "fatalities" : [],
        "Geometry" : []
    }

    for n in range(0, len(acled) - 1):

        # Get ACLED XY
        xy = acled.iloc[n]["geometry"]

        # Input X, Y, H3 resolution
        hexbin = h3.geo_to_h3(xy.x, xy.y, 5)

        # Calculate statistics to assign to hexbins
        if hexbin not in acled_to_h3.keys():
            acled_to_h3[hexbin] = {
                "count" : 1,
                "fatalities" : acled.iloc[n]["fatalities"],
                "Boundary" : h3.h3_to_geo_boundary(hexbin, geo_json=False)
             }
        else:
            acled_to_h3[hexbin]["count"] += 1
            acled_to_h3[hexbin]["fatalities"] += acled.iloc[n]["fatalities"]

    # Create dictionary for all hexbins and statistics to turn into DataFrame
    for key, val in acled_to_h3.items():
        h3_statistics["Hexbin"].append(key)
        h3_statistics["count"].append(val["count"])
        h3_statistics["fatalities"].append(val["fatalities"])
        h3_statistics["Geometry"].append(Polygon(val["Boundary"]))

    h3_geopandas = gpd.GeoDataFrame(h3_statistics)

    h3_geopandas.sort_values(by=theme)

    hexbins = h3_geopandas["Hexbin"].values.tolist()

    for hexbin in hexbins:

        count = h3_geopandas.loc[h3_geopandas["Hexbin"] == hexbin, theme].iloc[0]

        # Assign symbology based on NBJ valuess
        if count <= sorted_NBJ[0]:
            colour_ramp_choice = 0
        elif count > sorted_NBJ[0] and count <= sorted_NBJ[1]:
            colour_ramp_choice = 1
        elif count > sorted_NBJ[2] and count <= sorted_NBJ[3]:
            colour_ramp_choice = 2
        else:
            colour_ramp_choice = 3
        
        # Get hexbin geometry
        geom = h3_geopandas['Geometry'][h3_geopandas["Hexbin"] == hexbin]

        # Create Hexbin
        h3_UKR = ShapelyFeature(
            geom,
            map_crs,
            edgecolor="k",
            facecolor=colour_ramp[colour_ramp_choice]
        )

        # Plot hexbin
        map.add_feature(h3_UKR)
```

### **Create a series of graphs and maps to display these derived datasets**

This section of code relies upon three functions:

    1.	create_acled_graphs
    2.	create_oblast graphs
    3.	create_hexbin maps
    4.	calculate_centroids

The first three functions take all previously created datasets, which are either pandas or geopandas dataframes, and graphically or spatially plot them into graphs and maps.

These functions create new figures or map frames, and for each, the titles and surrounding figure elements are set, and the data is queried, plotted, and styled.

The calculate_centroids function calculates the centroid of each oblast, which it then uses as the XY coordinate the place the oblast label.

If alternative datasets or date ranges are to be used, then changes to the following sections will need to be made to work with the theme of choice.

For the create_acled_graphs function: 
	
    •	Titles
    •	Plotted data and columns
    •	x_ticks
    •	y_ticks
    •	Legend labels
    •	Vertical lines and labels for key dates

```python
def create_acled_graphs(acled_daily_statistics): 

    # Create figure and axis
    figure_1, (axis_1, axis_2, axis_3) = plt.subplots(nrows=3, ncols=1, figsize=(20,20))

    axis_list = [axis_1, axis_2, axis_3]

    # Set titles
    axis_1.set_title("ACLED : Daily Events and Fatalities",
                    fontsize=18, 
                    loc="left", 
                    weight="bold")

    axis_2.set_title("ACLED : Accumulative Events and Fatalities",
                    fontsize=18, 
                    loc="left", 
                    weight="bold")

    axis_3.set_title("ACLED : Daily Force Activity",
                    fontsize=18, 
                    loc="left", 
                    weight="bold")

    # Plot daily statistics by date
    axis_1.plot(acled_daily_statistics["date"], 
                acled_daily_statistics["count"], 
                color="yellow", 
                linestyle=":",
                label="Events ({})".format(acled_daily_statistics["count"].sum()))

    axis_1.plot(acled_daily_statistics["date"], 
                acled_daily_statistics["fatalities"], 
                color="r", 
                linestyle=":", 
                label="Fatalities ({})".format(acled_daily_statistics["fatalities"].sum()))

    axis_2.plot(acled_daily_statistics["date"], 
                acled_daily_statistics["count_sum"], 
                color="yellow", 
                linestyle=":",
                label="Events ({})".format(acled_daily_statistics["count_sum"].max()))

    axis_2.plot(acled_daily_statistics["date"], 
                acled_daily_statistics["fatalities_sum"],
                color="r", 
                linestyle=":",
                label="Fatalities ({})".format(acled_daily_statistics["fatalities_sum"].max()))

    axis_3.plot(acled_daily_statistics["date"], 
                acled_daily_statistics["ukraine_actor"], 
                color="blue", 
                linestyle=":",
                label="Ukrainian ({})".format(acled_daily_statistics["ukraine_actor"].sum()))

    axis_3.plot(acled_daily_statistics["date"], 
                acled_daily_statistics["russian_actor"], 
                color="red", 
                linestyle=":",
                label="Russian ({})".format(acled_daily_statistics["russian_actor"].sum()))

    axis_3.plot(acled_daily_statistics["date"], 
                acled_daily_statistics["other_actor"], 
                color="darkorange", 
                linestyle=":",
                label="Other ({})".format(acled_daily_statistics["other_actor"].sum()))

    # Set x and y axis labels
    x_ticks = [date_start, datetime(2022,2,1), datetime(2022,2,24), datetime(2022,4,1), date_end]  # UPDATE : Change for appropriate date values between date_start and date_end variables

    y_ticks = [[i for i in range(200, acled_daily_statistics["fatalities"].max() + 200, 200)], # UPDATE : ( Start value, max value, increment value )
            [i for i in range(1000, acled_daily_statistics["count_sum"].max() + 1000, 1000)], # UPDATE : ( Start value, max value, increment value )
            [i for i in range(20, 110, 20)]] # UPDATE : ( Start value, max value, increment value )

    # Format styling and legend
    for i, ax in enumerate(axis_list):
        ax.set_facecolor("lightslategray")
        ax.set_xticks(
            ticks=x_ticks, 
            labels=["01 Jan 2022", "01 Feb 2022", "24 Feb 2022", "01 Apr 2022", "15 Apr 2022"], # UPDATE : Change to match x_tick values
            fontsize=14
        )
        ax.set_yticks(ticks=y_ticks[i], labels=y_ticks[i], fontsize=14)
        ax.hlines(y=y_ticks[i], xmin=date_start, xmax=date_end, color="whitesmoke", linestyle=":")
        ax.vlines(
            x=mdates.date2num(datetime(2022,2,24)), # UPDATE : Vertical lines for key events
            ymin=0, 
            ymax=y_ticks[i][-1],
            color="k", 
            linestyle=":",
            alpha=0.5,
            label="Russian Invasion of Ukraine - 24 February 2022" # UPDATE : Name of key event
        )
        ax.legend(fontsize=14, loc="upper left", facecolor="slategray")
```

For the create_oblast_graphs function: 
	
    •	Titles
    •	Plotted data and columns
    •	y_ticks
    •	Extent for vertical lines

```python
def create_oblast_graphs(oblast_statistics):

    # Create figure and axis
    figure_2, (axis_4, axis_5) = plt.subplots(nrows=1, ncols=2, figsize=(60,20))

    axis_list = [axis_4, axis_5]

    # Reverse A-Z into Z-A for plot labels
    oblast_statistics = oblast_statistics.sort_values(by="Region", ascending=False)

    # Set titles
    axis_4.set_title("ACLED : Total Events by Ukranian Oblast (01 Jan 22 - 15 Apr 22)", # UPDATE : Change for dataset date range
                    fontsize=24, 
                    loc="left", 
                    weight="bold")

    axis_5.set_title("ACLED : Total Fatalities by Ukranian Oblast (01 Jan 22 - 15 Apr 22)", # UPDATE : Change for dataset date range
                    fontsize=24, 
                    loc="left", 
                    weight="bold")

    # Plot oblast_statistics
    axis_4.barh(oblast_statistics["Region"], oblast_statistics["count"], color="yellow", alpha=0.5)
    axis_5.barh(oblast_statistics["Region"], oblast_statistics["fatalities"], color="r", alpha=0.5)

    # Set y axis labels
    y_ticks = [[i for i in range(250, oblast_statistics["count"].max() + 250, 250)], [i for i in range(200, oblast_statistics["fatalities"].max() + 200, 200)]] # UPDATE : ( Start value, max value, increment value )

    # Format styling
    for index, axis in enumerate(axis_list):  
        axis.set_facecolor("lightslategray")
        axis.vlines(x=y_ticks[index], ymin="Autonomous Republic of Crimea", ymax="Zhytomyr region", color="whitesmoke", linestyle=":")
        axis.tick_params(axis='x', labelsize=18)
        axis.tick_params(axis='y', labelsize=18)

    for index, value in enumerate(oblast_statistics["count"]):
        axis_4.text(value + 20, index, str(value), color="k", fontsize=18, va="center")

    for index, value in enumerate(oblast_statistics["fatalities"]):
        axis_5.text(value + 2, index, str(value), color="k", fontsize=18, va="center")  
```

For the create_hexbin_maps function: 

    •	Titles

```python
def create_hexbin_maps(acled_daterange_sm, acled_daterange_me, acled_daily_statistics):

    """

    Create hexbins using Uber h3 geospatial indexing

    Parameters : 

        argument 1 (geopandas dataframe) : ACLED dataset for daterange start - mid
    
        argument 2 (geopandas dataframe) : ACLED dataset for daterange mid - end
        
        argument 3 (geopandas dataframe) : DataFrame for daily ACLED statistics


    Returns : 

        Creates various hexbin maps for fatalities and counts for the start - mid / mid - end data ranges

    """

    # List of map titles and associated DataFrames
    map_list = {
        "map_hex_count_1" : {"Title" : "ACLED : Event Count for 01 Jan 22 - 24 Feb 22", "GPD" : acled_daterange_sm}, # UPDATE : Change title to match dates
        "map_hex_count_2" : {"Title" : "ACLED : Event Count for 25 Feb 22 - 15 Apr 22", "GPD" : acled_daterange_me}, # UPDATE : Change title to match dates
        "map_hex_fatal_1" : {"Title" : "ACLED : Fatalities for 01 Jan 22 - 24 Feb 22", "GPD" : acled_daterange_sm}, # UPDATE : Change title to match dates
        "map_hex_fatal_2" : {"Title" : "ACLED : Fatalities for 25 Feb 22 - 15 Apr 22", "GPD" : acled_daterange_me} # UPDATE : Change title to match dates
        }
        
    for map_theme, vals in map_list.items():

        map = plt.figure(figsize=(20,30))

        map_crs = ccrs.Mercator()

        map_ax = plt.axes(projection=map_crs)

        # Set either count or fatalities theme
        if map_theme.startswith("map_hex_count"):
            theme = "count"
        elif map_theme.startswith("map_hex_fatal"):
            theme = "fatalities"

        # Calculate NBJ values for theme
        acled_daily_statistics["NBJ"] = pd.cut(
            acled_daily_statistics[theme],
            bins=jenkspy.jenks_breaks(acled_daily_statistics[theme], nb_class=4),
            include_lowest=True)

        set_NBJ = set(acled_daily_statistics["NBJ"].values.tolist())

        list_NBJ = list(set_NBJ)

        list_sort = []

        # List the max value for each NBJ range
        for i in list_NBJ:
            list_sort.append(int(i.right))

        list_sort.sort()

        leg_handle = []

        leg_lable = []

        # Create legend handles and labels based on theme and NBJ values
        for i, v in enumerate(list_sort):
            
            leg_handle.append(mpatch.Rectangle((0, 0), 1, 1, facecolor=colour_ramp[i], edgecolor="k"))

            if i == 0:
                leg_lable.append("{} : {} - {}".format(theme.capitalize(), 0, list_sort[i]))
            else:
                leg_lable.append("{} : {} - {}".format(theme.capitalize(), list_sort[i-1], list_sort[i]))

        map_ax.set_title(vals["Title"], 
                        fontsize=20, 
                        weight="bold",
                        loc="left")

        map_ax.add_feature(GLB_outline)

        # Create multiple buffers with opacity for visual effect
        create_buffers(buffer_values, transparency_values, map_ax)

        map_ax.legend(leg_handle, leg_lable, title="Legend", loc="upper left", framealpha=1, fontsize=14, title_fontsize=16)

        map_ax.add_feature(oblast_outline)
        map_ax.set_facecolor("skyblue")

        map_ax.set_extent([xmin - 3, xmax + 1, ymin - 1, ymax + 1], crs=map_crs)

        create_hexbins(vals["GPD"], map_ax, theme, list_sort, colour_ramp)
```

### **Run all functions**

Once all the above functions have been defined and set up correctly, the last part of the code calls all these functions to run, assigning new variables from the outputs in which to pass into the proceeding functions.

### **Results**

This tool outputs a series of graphs and maps  displaying various categorical, temporal, and spatial trends within the dataset. The deployed tool and associated datasets will produce the following OSINT products.

**Daily event statistics**

A series of lines graphs are created which display counts for the number of daily events and fatalities, events and fatalities accumulating over time, and a breakdown of daily events split by the responsible actors.


**Oblast statistics**

A pair of horizontal bar charts are created, showing the breakdown of total events and fatalities for each Ukrainian Oblast.

**Oblast events**

A choropleth map is created which displays values for the number of events within each oblast. Value ranges are calculated using natural breaks (Jenks), creating four classes based on natural groupings inherent in the data.

**Aggregated statistics**

Two pairs of maps are created, one for events and the other for fatalities, showing counts of aggregated data statistics temporally for the two date ranges; in the deployed tool this is pre-invasion and post-invasion of Ukraine. Data is aggregated using the Uber H3 hexagonal hierarchical geospatial indexing system at a resolution of 5 (approx. 252km2).