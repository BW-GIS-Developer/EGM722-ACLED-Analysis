## **Automate OSINT workflows with GIS**

## **Overview**

Open-source intelligence (OSINT) is the collection and analysis of open-source (OS) datasets to provide intelligence products. The Armed Conflict Location & Event Data Project (ACLED)[^1] collects real-time data on the locations, dates, actors, fatalities of violence, and protests around the world (ACLED, 2022). As each event has an associated location and date, GIS can be employed to spatially, and temporally analyse this dataset to extract and visualise trends for intelligence products.

The ”ACLED Analysis” GIS tool was developed to automate such a workflow using the publicly available ACLED dataset; it will focus on the current conflict within Ukraine. It will detail incident and fatality counts over time, determine the most affected regions, and spatially plot the temporal patterns of the conflict.

[^1]: The Armed Conflict Location & Event Data Project (ACLED); [http://acleddata.com](http://acleddata.com)

## **Setup and Installation**

The code for this GIS tool is publicly hosted on GitHub and can be accessed from the EGM722-ACLED-Analysis repository; [https://github.com/BW-GIS-Developer/EGM722-ACLED-Analysis.git](https://github.com/BW-GIS-Developer/EGM722-ACLED-Analysis.git).

The code was developed using Jupyter Notebooks and Visual Studio Code and should be run using an integrated development environment (IDE) that can support the .ipynb file type; It is recommended that Anaconda Navigator[^2]  is used for this.

Several python dependencies are needed for the code to run, these are detailed in table 1. These dependencies should be installed into a virtual environment by using the [environment.yml](https://github.com/BW-GIS-Developer/EGM722-ACLED-Analysis/blob/main/environments.yaml) file saved in the GitHub repository; this requires Anaconda to be installed as it uses conda-forge to install and saves the virtual environment within the Anacoda3 directory. Alternatively, they can be installed individually using the conda-forge channel from within the conda command prompt.

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

[^2]: Anaconda Navigator; [https://docs.anaconda.com/anaconda/install/](https://docs.anaconda.com/anaconda/install/)
[^3]: ACLED export filters applied - From: 01 Jan 22, To: 15 Apr 22, Region: Europe, Country: Ukraine

## **Methodology**