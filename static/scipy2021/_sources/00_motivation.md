# Overview

The yt package is a cross-domain tool for visualization and analysis of data. While still used primarily within the computational astrophysics community, a number of recent efforts have focused on transforming yt into a more domain agnostic tool. Part of the effort to expand the yt platform includes leveraging recent advancements from across the open source python ecosystem. This presentation focuses on advancements in using Dask within the yt framework to improve both serial and parallel workflows.

While yt already supports lazy serial and MPI-parallel computations, refactoring yt to use Dask has a number of potential benefits. For the yt user, an underlying Dask framework allows for parallel computation with minimal setup and a consistent experience across machines from laptops to HPC environments. It also allows for easier custom parallel workflows, for example by returning Dask arrays rather than MPI iterable objects (while still operating in MPI environments with dask-mpi) from which a user can use more array-like manipulations while still working in parallel. For the developer, a Dask refactor will simplify the codebase and improve the development process, particularly for implementing new parallel algorithms.

In this presentation, we will show our latest efforts to leverage Dask within the yt framework, building off of previous reports on Dask-yt prototyping shown at RHytHM2020 ([Leveraging Dask in yt](https://youtu.be/3GLbEBgpaK4)) and the yt-blog ([Dask and yt: a pre-YTEP](https://yt-project.github.io/blog/posts/dask_yt_pytep/)). These reports described a number of separate experimental prototypes including a "Daskified" particle reader, a binned-statistic calculation using yt's already-optimized parallel statistic methods with Dask delayed arrays and unyt arrays with Dask support. In this presentation, we will present a more fully integrated prototype directly within the yt API. We will show comparisons in use and performance between the existing chunk framework and the new Daskified versions for both single-processor serial and parallel computations. 

The poster is divided into a number of sections:

* [Daskified unyt arrays](01_unytdask): demonstrating a daskified version of the arrays underlying yt.
* [Daskified reads](02_daskifiedread): a discussion of yt IO and an implementaiton of a daskified particle reader for yt. 
* [Daskified computations](03_daskifiedquantity): demonstrating some daskified reductions.
* [Dasken(yt)](04_finalthoughts): some final thoughts on further Daskening. 
* [Repository notes](99_setup): notes on the code used in this poster.