---
title: "yt_xarray updates at the 2024 Software for the NASA SMD Workshop"
date: 2024-05-30T13:55:49-06:00
tags: ["code", "xarray", "yt", "geodata", "yt-xarray", "volume_rendering"]
categories: ["updates"]
---

I recently had the pleasure of presenting updates to `yt_xarray` at the 2024 Software for the NASA SMD Workshop. Read on to check out the full presentation.

## yt-xarray, Facilitating Software Reuse Between Space and Earth Sciences

My presentation covered our recent efforts with `yt_xarray` to link `yt` and `xarray`. At it's most basic, `yt_xarray` is a glue-package that facilitates data transfer between the two packages. But it's got some bells and whistles attached to streamline that data communication and to simplify some of the `yt` usage for geospatial volumetric data.

You can check out the full presentation (which I wrote as a Jupyter book) in a number of ways:
* [online rendering](https://chrishavlin.github.io/NASASoftwareWorkshop2024/intro.html)
* PDF rendering from the zenodo repository at https://doi.org/10.5281/zenodo.11121692

This presentation included examples with the new embedded transformations framework, which has since been released in `yt_xarray` v0.3.0 just last week ([v0.3.0 release notes](https://github.com/data-exp-lab/yt_xarray/releases/tag/v0.3.0)).
