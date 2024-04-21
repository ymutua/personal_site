---
title: Extract weather and climate data from online repositories
author: John Mutua
date: '2019-09-13'
slug: []
categories:
  - GIS
  - R
  - maps
tags:
  - GIS
  - maps
  - R
subtitle: ''
excerpt: ''
series: ~
layout: single
---



In a world experiencing climate change, good weather and climate data matters. It may be available but often hard to find, understand and apply to decision-making mainly because:

- There are no weather stations everywhere
- Weather stations are not in good condition (gaps)
- The data is not stored correctly
- The data does not pass the basic quality controls
- Access to data is restricted

However, weather and climate data products derived from satellites and model-based reanalysis have been used in locations where station based observations are inadequate or incomplete.

We will explore methods of extracting weather and climate data from [NASA Power](https://power.larc.nasa.gov/data-access-viewer/)

If you chose to extract data from NASA Power, have your coordinates ready then visit the NASA Power [website](https://power.larc.nasa.gov/data-access-viewer/) and download manually.

![](images/nasa_power.png)
However, this can be painful if you have alot of sites. The solution is to develop custom scripts to automate downloading. For this you'll need:

- R
- RStudio (*optional*)
- Packages to make easier management of the data
- Your list of stations with coordinates stored as a `.csv` file

We will extract two parameters i.e. precipitation and temperature, on a daily time step. For more information about other parameters nand temporal resolutions available see [here](https://power.larc.nasa.gov/docs/v1/). The API allows you to:

- Make HTTPS GET calls to the POWER data archives directly.
- Integrate the service into your own applications.

For a sample SinglePoint, [here](https://power.larc.nasa.gov/cgi-bin/v1/DataAccess.py?request=execute&identifier=SinglePoint&parameters=T2M,PS,ALLSKY_SFC_SW_DWN&startDate=20160301&endDate=20160331&userCommunity=SSE&tempAverage=DAILY&outputList=JSON,ASCII&lat=36&lon=45&user=anonymous) is the call to the API.

The `nasapower` R package has been developed for this! It makes it easy to:

- Automate downloading from NASA-POWER website.
- Generate input data for use in crop models like [APSIM](http://www.apsim.info/) or [DSSAT](https://dssat.net/).

To pull daily data from the API for a single coordinate, you will first need to load `nasapower` R package through , then use the `get_power` function as shown below.


```r
daily_ag <- nasapower::get_power(community = "ag",
                      lonlat = c(43.13333, 11.55),
                      pars = c("T2M", "PRECTOTCORR"),
                      dates = "2010-01-01",
                      temporal_api = "daily"
                      )
```

What about many sites? i.e `sites > 1`. For this, you will need a bunch of libraries besides `nasapower` R package.


```r
library(nasapower)
library(data.table)
library(tidyverse)
```

Next, create your own station data.


```r
ID <- c("63125", "63021", "63023")
Lat <- c(11.55, 15.283333, 15.616667)
Long <- c(43.133333, 38.916667, 39.45)

sites <- data.frame(ID, Lat, Long)
```

Create a list to store your data.


```r
daily_ag <- list()
```

Using a `for` loop, download for each station.


```r
for(i in 1:nrow(sites)){

  site_row<- sites[i,]

  daily_ag[[i]] <- nasapower::get_power(community = "ag",
                             lonlat = c(site_row$Long, site_row$Lat),
                             pars = c("T2M", "PRECTOTCORR"),
                             dates = c("2008-01-01", "2018-12-31"),
                             temporal_api = "daily")
  
  site_id = site_row$ID 
  
  daily_ag[[i]]$site_id <- site_id

}
```

Create one data table from the list saved above.


```r
data <- rbindlist(daily_ag)
```

We can do some visualization. But first let's group the data by `site_id` then `YEAR` using pipes: *read as, and then*.


```r
data_prec <- data %>%
  group_by(site_id, YEAR) %>%
  summarize(annual_prec = sum(PRECTOTCORR, na.rm = TRUE))
```

The dataset has three variables that need fixing.


```r
# color and split variables
data_prec$site_id <- as.factor(data_prec$site_id)
data_prec$YEAR <- as.factor(data_prec$YEAR)
```

Plot annual precipitation.


```r
ggplot(data_prec, aes(x = YEAR, 
                           y = annual_prec, 
                           color = site_id,
                           group = site_id)) +
  geom_line()+
  geom_point()+
  ggtitle("Average Annual Precipitation") +
  xlab("Year") + ylab("Annual Precipitation") +
  theme(plot.title = element_text(hjust = 0.5))
```

<img src="{{< blogdown/postref >}}index_files/figure-html/annual_prec-1.png" width="90%" />
