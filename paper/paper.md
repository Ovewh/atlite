---
title: 'atlite: A Lightweight Python Package for Calculating Renewable Power Potentials and Time-Series'
tags:
  - Python
  - energy systems
authors:
  - name: Fabian Hofmann
    orcid: 0000-0002-6604-5450
    affiliation: "1" # (Multiple affiliations must be quoted)
    # add your names here
  - name: Johannes Hampp
    orcid: 0000-0002-1776-116X
    affiliation: "2"
affiliations:
 - name: Frankfurt Institute for Advanced Studies
   index: 1
 - name: Center for international Development and Environmental Research (ZEU), Justus-Liebig University Giessen
   index: 2
 - name: Institution Name
   index: 3
date: 01 March 2021
bibliography: paper.bib

# Optional fields if submitting to a AAS journal too, see this blog post:
# https://blog.joss.theoj.org/2018/12/a-new-collaboration-with-aas-publishing
aas-doi: 10.3847/xxxxx #<- update this with the DOI from AAS once you know it.
aas-journal: xxxx #<- The name of the AAS journal.
---

<!-- See https://joss.readthedocs.io/en/latest/submitting.html for all details -->
<!-- compile with 

pandoc --citeproc -s paper.md -o paper.pdf  

 -->

# Summary
<!-- Change whatever you want -->

Renewable energy sources build the backbone of the future global energy system. For a successful energy transition it is crucial to rigorously analyse the weather-dependent energy outputs of eligible and existent renewable resources. atlite is an open python software package for retrieving reanalysis weather data and converting it to potentials and time series for renewable energy systems. Based on detailed mathematical models, it simulates the power output of wind turbines, solar photo-voltaic panels, solar-thermal collectors, run-of-river power plants and hydro-electrical dams. It further provides weather-dependant projections on the demand side like heating demand degree days and heat pump coefficients of performance.


# Statement of need


<!-- context of atlite -->
<!-- FABIAN NEUMANN -->
atlite was initially build as a light-weight alternative to the Danish REatlas [@andresen_validation_2015]. Downsides of REatlas.  What other packages exist?  What's their up/downside? Why is atlite necessary? 

<!-- software/packages and implementation -->
<!-- JOHANNES -->
Deriving weather-based time-series and potentials for renewables over large regions is a common problem in energy system modelling.
Websites with exposed open APIs such as [renewables.ninja](https://www.renewables.ninja) [@pfenninger_long-term_2016][@staffell_using_2016] for such purpose exist but are difficult to use for local execution in e.g. cluster environments.
Further they expose by-design neither the underlying datasets nor methods for deriving time-series thus making them unsuited for exploring alternative weather-to-time-series conversion methods or utilising different weather datasets.
Other libraries like [pvlib](https://github.com/pvlib/pvlib-python) [@holmgren_pvlib_2018] are suited for local execution and allow exchangeable input
data but are specialised to certain renewables (PV systems in this case) and intended for single location modelling.

The purpose of atlite is to fill this gap and provide a library to derive time-series and potentials for renewables based on weather datasets.
atlite is designed with extensibility for new types of renewables or different time-series conversion models in mind.
An abstraction layer for weather datasets enables flexibility for exchange of the underlying datasets.
By leveraging the Python packages [xarray](https://xarray.pydata.org/en/stable/) [@hoyer_xarray_2017],
[dask](https://docs.dask.org/en/latest/) [@dask_2016] and [rasterio](https://rasterio.readthedocs.io/en/latest/) atlite makes use of parallelisation
and memory efficient backends thus scaling well on even large datasets.

# Basic Concept


The starting point of most atlite functionalities  is the `atlite.Cutout` class. It serves as a container for a spatio-temporal subset of one or more weather datasets. As illustrated below, a typical workflow consists of three steps: Cutout creation, Cutout preparation and Cutout conversion. 

![Typical working steps with `atlite`. \label{fig:cutout}](figures/workflow.png)




## Cutout Creation and Preparation
<!-- FABIAN H -->


The Cutout creation requires the following arguments: geographical and temporal bounds, the path of the associated `netcdf` file (which will be created) as well as the source for the weather data, refered to as module. Optionally, temporal and spatial resolution may be adjusted, the default is set to 1 hour and 0.25$^\circ$ latitude times 0.25$^\circ$ longitude. So far, atlite supports three weather data sources: 

1. [ECMWF Reanalysis v5 (ERA5)](https://www.ecmwf.int/en/forecasts/dataset/ecmwf-reanalysis-v5) provides various weather-related variables in an hourly resolution from 1950 onward on a spatial grid with a 0.25$^\circ$ x 0.25$^\circ$ resolution. Atlite automatically retrieves the raw data using the [Climate Data Store (CDS) API](https://cds.climate.copernicus.eu/#!/home) which has to be properly set up by the user. When the requested data points diverge from the original grid, the API retrieves averaged values based on the original grid data **(double check)**. 

2. [Heliosat (SARAH-2)](https://wui.cmsaf.eu/safira/action/viewDoiDetails?acronym=SARAH_V002) provides satellite-based solar data in a 30 min resolution from 1983 to 2015 on a spatial grid from -65° to +65$^\circ$ longitude and latitude in a 0.05$^\circ$ x 0.05$^\circ$ resolution. The dataset must be downloaded by the user beforehand. By using a re-gridding function provided by atlite, the data may be projected on arbitrary resolutions. 

3. [GEBCO](https://www.gebco.net/data_and_products/gridded_bathymetry_data/) is a bathymetric data set covering terrain heights on a 15 arc-second resolved spatial grid. The dataset has to be downloaded by the used beforehand. 

When initializing a Cutout, the grid cells and the coordinate system on which the data will lay are created. As indicated in the above figure, the shapes of the weather cells are created such that their coordinates are centered in the middle. As soon as the preparation of the cutout is executed, atlite retrieves/loads data variables, adds them to the Cutout and finally writes the Cutout out to the associated netcdf file. 
`Atlite` groups weather variables into *features*, which can be used as front-end keys for retrieving a subset of the available weather variables. The following table shows the variable groups for all datasets.


|   feature   |                    ERA5 <br/> variables                    |        SARAH-2<br/> variables         | GEBCO variables |
| :---------- | :--------------------------------------------------- | :------------------------------- | :-------------- |
| height      | height                                               |                                  | height          |
| wind        | wnd100m, roughness                                   |                                  |                 |
| influx      | influx\_toa, influx\_direct,<br/>influx\_diffuse, albedo | influx\_direct,  influx\_diffuse |                 |
| temperature | temperature, soil temperature                        |                                  |                 |
| runoff      | runoff                                               |                                  |                 |


A Cutout may combine features from different sources, e.g. 'height' from GEBCO and 'runoff' from ERA5. Future versions of atlite will likely introduce the possibility to retrieve explicit weather variables from the CDS API. Further, the climate projection dataset [CORDEX](https://rcmes.jpl.nasa.gov/content/cordex) which was removed in v0.2 due to compatibility issues, is likely to be reintroduced. 


## Conversion Functions


Atlite currently offers conversion functions for deriving time-series and static potentials from cutouts for the following types of renewables:

* Solar photovoltaics:
Two alternative solar panel models based on [@huld_mapping_2010] and [@beyer_robust_2004]
using the clearsky model from [@reindl_diffuse_1990],
solar azimuth and altitude position tracking based on [@michalsky_astronomical_1988,@sproul_derivation_2007,@kalogirou_solar_2009] combined with a surface orientation algorithm following
[@sproul_derivation_2007] and optimal latitude heuristics from [@landau_optimum_2017].

* Solar thermal collectors:
An implementation for providing low temperature heat for space or district heating from
[@henning_comprehensive_2014] which combines average global radiation with and storage losses
dependent on the current outside temperature.

* Wind turbines:
A general implementation of wind turbine power output using included or arbitrary power curves
as well as optional convolution with a Gaussian kernel for region specific calibration given
real-world reference data as presented by [@andresen_validation_2015].

* Hydro run-off power:
An heuristic approach which uses runoff weather data which is normalised to match reported 
energy production figures by the [EIA](https://www.eia.gov/international/data/world).
The resulting time-series are optionally weighted by height of the runoff location and time-series may
be smoothed for a more realistic representation.

* Hydro reservoir and dam power:
Following [@liu_validated_2019] and [@lehner_global_2013] run-off data is aggregated to
and collected in basins which are obtained and estimated in their size with the help
of the [HydroSHEDS](https:// hydrosheds.org/) dataset.

* Heating demand:
Space heating demand is obtained with a simple degree-day approximation where
the difference between outside ground-level temperatures and a reference temperature
scaled by a linear factor yields the desired estimate.



The conversion functions are highly flexible and allow the user to calculate different types of outputs, which arise from the set of input arguments. In energy system models, network nodes are often associated with geographical regions which serve as catchment areas for electric loads, renewable energy potentials etc. As indicated in third step of the above figure, `atlite`'s conversion functions allow to project renewable time-series on a list of bus regions. Therefore, `atlite` internally computes the Indicator Matrix $\textbf{I}$ with values $I_{r,x,y}$ representing the per-unit overlap between bus region $r$ and the weather cell at $(x,y)$. Further, the user can weigh the weather cells 



## Land-Use Availability
<!-- FABIAN HOFMANN -->

In the real world, renewable infrastructure is often limited by land-use restrictions. Wind turbines can only be placed in eligible places which, according to the country specific policy, has to fulfill certain criteria, e.g. non-protected areas, enough distance to residential areas etc. `Atlite` provides a performant, parallelized implementation to calculate land-use availabilities within all weather cells of a cutout. The user can exclude geometric shapes and/or geographic rasters of arbitrary projection, like the [Corine Land Cover (CLC)](https://land.copernicus.eu/pan-european/corine-land-cover), from the eligible area.
The implementation is inspired by the [GLAES](https://github.com/FZJ-IEK3-VSA/glaes) [@Ryberg2018]
software package which itself is no longer maintained and incompatible with newer versions of the underlying [GDAL](https://gdal.org/index.html) software.

![Example of a land-use restrictions calculated with atlite. The left side shows a highly-resolved raster with available areas in green. Excluded areas, which in this example are set to all urban and forest-like sites, are drawn in white. Resulting entries of the availability matrix computed by `atlite` is shown on the right hand side.  \label{fig:land-use}](figures/land-use-availability.png)



# Related Research 
<!-- FABIAN NEUMANN -->


atlite is used b several research projects and groups. The open-source PyPSA-EUR workflow [**cite**] is a   




# Availability
<!-- WHO EVER WANTS -->

Stable versions of the atlite package are available for Linux, MacOS and Windows via
`pip` in the [Python Package Index (PyPI)](https://pypi.org/project/atlite/) and
for `conda` on [conda-forge](https://anaconda.org/conda-forge/atlite).
Upstream versions and development branches are available in the projects [GitHub repository](https://github.com/PyPSA/atlite).
Documentation including examples are available on [Read the Docs](https://atlite.readthedocs.io/en/latest/).
The atlite package is released under [GPLv3](https://github.com/PyPSA/atlite/blob/master/LICENSES/GPL-3.0-or-later.txt) and welcomes contributions via the projects [GitHub repository](https://github.com/PyPSA/atlite).


# Acknowledgements
<!-- WHO EVER WANTS -->


# References