# yt_xarray: Facilitating Software Reuse Between Space and Earth Sciences

## Authors

* Chris Havlin: University of Illinois Urbana-Champaign, School of Information Sciences,
* Matt Turk: University of Illinois Urbana-Champaign, School of Information Sciences, 

## Abstract

In this presentation, we describe recent efforts to better link two 
open source Python packages, yt and xarray, in order to improve the ability to 
use yt with NASA, cloud-hosted datasets. yt provides support for the analysis and 
visualization of multi-resolution volumetric data. It can read, understand, and 
process data from dozens of different platforms, including well-used and 
established astrophysical simulation data formats as well as observational and 
simulation data from a number of geophysical domains. Xarray is a popular 
metadata-preserving array library with excellent support for analysis of remotely 
stored data, particular through its support for cloud-native formats like Zarr. 
Recent work on both a new yt_xarray package and core yt has simplified the
ability to analyze and visualize geospatial and geophysical datasets loaded 
with xarray using yt. In this presentation, we describe some of the improvements, 
including 3D volume rendering with embedded interpolation and use of yt analysis
methods with remotely hosted data using xarray as a data backend.

