---
title: The neutralNEMO library
summary: A library for calculating neutral surfaces in NEMO
date: "2023-12-21T00:00:00Z"

reading_time: false  # Show estimated reading time?
share: true  # Show social sharing links?
profile: true  # Show author profile?
comments: false  # Show comments?

# Optional header image (relative to `static/img/` folder).
header: 
  caption: ""
  image: ""
---
<p style="text-align: center;">
{{< svg "static/img/pythonlogo.svg" >}}
</p>

neutralNEMO is a Python library which helps users calculate neutral surfaces from NEMO model outputs.

* The **GitHub repository** can be found here: [https://github.com/afstyles/neutralNEMO](https://github.com/afstyles/neutralNEMO)

* **Documentation** including installation instructions can be found here: [https://neutralnemo.readthedocs.io/en/latest/](https://neutralnemo.readthedocs.io/en/latest/)

The library makes it easier to use the highly-optimized algorithms of the [neutralocean](https://github.com/geoffstanley/neutralocean) package ([Stanley et al., 2021](https://doi.org/10.1029/2020MS002436)). 

The package contains routines to correctly load temperature and salinity date alongside the necessary horizontal and vertical grid information. There is also a streamlined routine for calculating an approximately neutral density surface (an omega surface) which is fixed to a specific location in time and space. Additional options to calculate the Veronis density and potential density surfaces are included.



***
**Note:** neutralNEMO is currently in **pre-release** and at an early stage of development. I am keen to hear how others get on with the library when they use it on their model data. If you encounter any issues please let me know or raise an issue on GitHub, I will be happy to help!
***
 
