---
layout: page
title: "Create Regions"
description: "tips on creating reporting units"
category: setup
tags: [python, arcgis, gis, tutorial]
---
{% include JB/setup %}

The first step in a custom Ocean Health Index analysis is creating the reporting units for which you will summarize. For a country level analysis, this is most likely to be the states or provinces, i.e. the first administrative level below country.

Here we provide a little recipe for creating these in ArcGIS using the following data:
* **GADM** - the Global Administrative Areas ([gadm.org](http://gadm.org)) contain the geography up to 5 political levels for all countries globally. 
* **EEZ** - the Exclusive Economic Zones ([marineregions.org](http://marineregions.org)) describe the 200 nautical mile (nm) extent of all countries.
* **EEZ and land** - with inclusion of the land and EEZ ([marineregions.org](http://marineregions.org)), we can fill in any gaps between GADM and the EEZ.

## Issues

There are several technical issues in generating regions which need need to: 1) not overlap with one another; and 2) cover the entire extent of the country without gaps. Here is the raw data displayed for our example country of Malaysia.

![raw data: GADM and EEZ](/figs/create_regions/gadm.png)

The EEZ is in blue and the GADM is color coded by state. Notice finer subdivisions are included in the GADM. Now we just need to extend these subdivisions offshore to generate state-level waters. The typical [buffer](http://resources.arcgis.com/en/help/main/10.2/index.html#//000800000019000000) function in GIS, however, does not handle distinct overlapping regions.

![raw data: GADM and EEZ](/figs/create_regions/overlapping.png)

As you can see from this simple buffer result, the buffers extend into each other. We solve for this overlap issue by generating [Thiessen polygons](http://resources.arcgis.com/en/help/main/10.2/index.html#//00080000001m000000) from the points on the outer edge of the land, which then gets intersected with the dissolved buffer. This is akin to the [method](http://marineregions.org/eezmethodology.php) used to originally create the EEZ boundaries. The result in this case with multiple buffers (at 3, 12, 50 and 100 nm) is unique and non-overlapping by state.

![raw data: GADM and EEZ](/figs/create_regions/buffers.png)

Other issues become apparent if we zoom in. There is a clear mismatch in the land described by GADM versus assumed by the EEZ. We presume that the EEZ is authoritative and therefore need to correct for the land.

![raw data: GADM and EEZ](/figs/create_regions/issues.png)

By extending the GADM provinces out with these Thiessen polygons to the full extent of the EEZ and intersecting with the missing EEZ land, hitherto missing land can be attributed to a state.

![raw data: GADM and EEZ](/figs/create_regions/corrected.png)

Some manual editing may be required beyond this recipe, since certain islands are not likely to be intersected by states and should instead by assigned wholly to one state. If the given country spans the international dateline (-180&deg; W or 180&deg; E), then a more complex analysis using geodetic distance should first be applied to the points (eg using [geographiclib](http://code.env.duke.edu/projects/mget/ticket/549)) and probably a [Plate Carr&eacute;e](http://resources.arcgis.com/en/help/main/10.2/index.html#//003r0000003r000000) or other dateline spanning projection should be used. Finally, if the country's EEZ is extensive than this method could be replaced with a raster method ("raster is faster, but vector is corrector") using [Polygon to Raster](http://resources.arcgis.com/en/help/main/10.2/index.html#//001200000030000000) and [Euclidean Allocation](http://resources.arcgis.com/en/help/main/10.2/index.html#//009z0000001m000000) (alternate recipe forthcoming).

## Script Recipe

Here's a script which you can modify based on your local paths and desired buffer distances. It is recommended that each line be run sequentially in the [Python window of ArcMap](http://resources.arcgis.com/en/help/main/10.2/index.html#/What_is_the_Python_window/002100000017000000/) which will render the geographic outputs so you can visually inspect the process.

{% gist 7650602 create_regions.py %}

## Future Work
* Fold this script into a function as part of the ohi-arcgis and ohi-opengis modules.
* Create a global base layer extending the finest GADM subdivision and desktop functions to operate on this layer, which will greatly reduce the processing time (by removing the Create Thiessen Polygons step).
* Create a web service for extraction of any country and subdivision buffer offshore or inshore.
* Create a custom [Albers Equal Area](http://resources.arcgis.com/en/help/main/10.2/index.html#//003r0000001n000000) projection assigning the parallels to 1/6th of the EEZ extent to minimize distortion (a la [project_optimal_albers.py](http://code.env.duke.edu/projects/mget/attachment/ticket/231/project_optimal_albers.py)).
* generate [TopoJSON](https://github.com/mbostock/topojson) for display in the OHI Toolbox mapping interface which minimizes the storage size by removal of redundant vertices of polygon shared borders. (Bonus: [Github interactive map rendering](http://blog.thematicmapping.org/2013/06/converting-shapefiles-to-topojson.html).)