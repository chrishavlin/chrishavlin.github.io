---
title: "Using moduli calculated from Abers and Hacker 2016 in the VBRc"
date: 2024-01-04T13:52:50-06:00
draft: false
---

The most recent feature release of the VBRc (version 1.1) included some changes that
simplify the specification of the unrelaxed moduli used within the VBRc. These changes
allow you to directly specify the unrelaxed shear and bulk moduli at the temperature and
pressure of interest and have those values automatically used in the calculation of
relaxed moduli. This post demonstrates how to use this new feature by using the MATLAB
code from Abers and Hacker 2016 for calculation anharmonic shear and bulk moduli and the
density.

## Background: anharmonic calculations and the VBRc

The default anharmonic calculation in the VBRc has always been intentionally simple. There
has been a significant amount of work in the geophysics community on approaches to calculating
anharmonic moduli, and so the VBRc initially focused on the anelastic calculations and included
only a simple linear scaling of anharmonic moduli. Comparing predictions from the VBRc to observed seismic properties, however, often requires calculating properties over large temperature and pressure ranges and the linear anharmonic scaling can introduce errors over
these large ranges.

To overcome these limitations, Josh Russel (link) recently worked on a pull request that to allow the VBRc to accept
inputs for the shear and bulk moduli more easily. It is intentionally open-ended: rather than adding a new
calculation the the VBRc, you can instead leverage other codes in the community and simply read
in the unrelaxed moduli as needed. But in addition to reading from files, the method for obtaining the anharmonic moduli could be your own custom calculation, some other MATLAB code.  In his work, Josh pre-computes pressure-temperature look up tables using Perplex (link) and then reads those values in when calling the VBRc. In the example here, I'll walk through a different workflow that is confined completely to MATLAB: using the Abers and Hacker 2016 toolbox to calculate the unrelaxed moduli and density at a range of pressures and temperatures and then feeding those results into the VBRC as inputs.

## Abers and Hacker 2016

overview

## The demo

The full code is on github in [this repository](https://github.com/chrishavlin/vbrc_anharmonic_experiments). You can check out the repository README for a full description
of how to setup and run the example, but I'll walk through the main features here.

## References
* Josh AGU poster?
* Abers and Hacker
* VBRc citation
* JF10
