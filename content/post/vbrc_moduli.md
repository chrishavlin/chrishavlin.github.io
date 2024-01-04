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

## Background: anharmonic properties and the VBRc

The default anharmonic calculation in the VBRc has always been intentionally simple. Given (1) the significant amount of work in the geophysics community on approaches to calculating
anharmonic moduli (2) the VBRc's focus on anelasticity, the VBRc includes
only a simple linear scaling of anharmonic moduli. Comparing predictions from the VBRc to observed seismic properties, however, often requires calculating properties over large temperature and pressure ranges and the linear anharmonic scaling can introduce errors over these large ranges. Additionally, while the VBRc's anelastic methods do not generally apply to compositions other than olivine at present, when comparing VBRc output to observations it can be helpful to include compositional variations to, for example, include compositional variations in the shallow lithosphere where intrinsic anelastic effects are negligible on seismic timescales.

To overcome these limitations, [Josh Russel](https://github.com/jbrussell) recently worked on a [pull request](https://github.com/vbr-calc/vbr/pull/63) to allow the VBRc to accept
inputs for the shear and bulk moduli more easily. The implementation adds new input fields to the VBR structure so that you can set the
shear and bulk moduli at your thermodynamic state in any way you choose. In Josh's case, this involves using [Perple_X](https://www.perplex.ethz.ch/perplex_66_seismic_velocity.html) to pre-compute pressure-temperature look up tables that then get read sampled before calling the VBRc, but you could instead leverage some other community code or write your own function to use. In the example here, I'll walk through a workflow that is confined completely to MATLAB: using the Abers and Hacker 2016 database and accompanying toolbox to calculate the unrelaxed moduli and density at a range of pressures and temperatures and then feeding those results into the VBRc as inputs.

## Abers and Hacker 2016

overview

## The demo

The full code is on github in [this repository](https://github.com/chrishavlin/vbrc_anharmonic_experiments). You can check out the repository README for a full description
of how to setup and run the example, but I'll walk through the main features here.

## References
* Abers and Hacker
* VBRc citation
* JF10
