# Create and plot timeseries

> notebook filename | 05-timeseries\_chl.Rmd  
> history | converted to R notebook from Timeseries\_CHL.R

This example extracts a time-series of monthly satellite chlorophyll
data for the period of 1997-present from four different monthly
satellite datasets:

  - SeaWiFS, 1997-2012 (ID = erdSWchlamday)  
  - MODIS, 2002-present (ID = erdMH1chlamday)  
  - VIIRS, 2012-present (ID = nesdisVHNSQchlaMonthly)  
  - OC-CCI, 1997-2017, a blended product that merges multiple satellite
    missions (ID = pmlEsaCCI31OceanColorMonthly)

The exercise demonstrates the following techniques:

  - Using **xtracto\_3D** to extract data from a rectangular area of the
    ocean over time
  - Using **rerddap** to retrieve information about a dataset from
    ERDDAP
  - Comparing results from different sensors  
  - Averaging data temporally and spatially
  - Producing scatter plots  
  - Producing line plots  
  - Drawing maps with satellite data using
**ggplot**

## Install required packages and load libraries

``` r
# Function to check if pkgs are installed, install missing pkgs, and load
pkgTest <- function(x)
{
  if (!require(x,character.only = TRUE))
  {
    install.packages(x,dep=TRUE,repos='http://cran.us.r-project.org')
    if(!require(x,character.only = TRUE)) stop(x, " :Package not found")
  }
}

# create list of required packages
list.of.packages <- c( "rerddap","plotdap", "lubridate", "maps",
                       "mapdata", "maptools", "mapproj", "ncdf4",
                       "reshape2", "colorRamps", "sp", "plyr", 
                       "ggplot2", "gridExtra", "rerddapXtracto")

# create list of installed packages
pkges = installed.packages()[,"Package"]

for (pk in list.of.packages) {
  pkgTest(pk)
}

# create list of required packages not installed
#new.packages <- list.of.packages[!(list.of.packages %in% pkges)]

# install missing packages
#if(length(new.packages)) install.packages(new.packages)
```

## Define the area boundaries

Next, extract data for an area in the Southern California Bight, between
-120 to -115 degrees east longitude and 31 to 34 degrees north latitude.

  - Set the longitude range: xcoord\<-c(-120, -115)  
  - Set the latitude range: xcoord\<-c(31,34)

<!-- end list -->

``` r
xcoord<-c(-120, -115)
ycoord<-c(31,34)

##Format Box Coordinates for cosmetics, to make a nice map title
ttext<-paste(paste(abs(xcoord), collapse="-"),"W, ", paste(ycoord, collapse="-"),"N")
```

## Extract satellite data with rxtracto\_3D

For each dataset, you will extract satellite data for the entire length
of the available timeseries.

  - Dates must be defined separately for each dataset. **rxtracto\_3D**
    will crash if dates are entered that are not part of the
    timeseries.  
  - The beginning (earliest) date to use in timeseries is obtained from
    the information returned in dataInfo.  
  - To get the end (most recent) date to use in the timeseries, use the
    `last` option for time.

### Get the SeaWiFS data

To begin, examine the metadata for the SeaWiFS monthly dataset (ID =
erdSWchlamday). The script block below:

  - Gathers information about the dataset (metadata) using **rerddap**  
  - Displays the information

<!-- end list -->

``` r
# Use rerddap to get information about the dataset
url="https://coastwatch.pfeg.noaa.gov/erddap/"
dataInfo <- rerddap::info('erdSWchlamday', url=url)

# Display the dataset metadata
dataInfo
```

    ## <ERDDAP info> erdSWchlamday 
    ##  Base URL: https://coastwatch.pfeg.noaa.gov/erddap/ 
    ##  Dataset Type: griddap 
    ##  Dimensions (range):  
    ##      time: (1997-09-16T00:00:00Z, 2010-12-16T12:00:00Z) 
    ##      altitude: (0.0, 0.0) 
    ##      latitude: (-90.0, 90.0) 
    ##      longitude: (0.0, 360.0) 
    ##  Variables:  
    ##      chlorophyll: 
    ##          Units: mg m-3

**Set the arguments for the data extraction**

  - **rxtracto3D** will need you to provide the parameter and
    coordinates for the extraction
  - For the parameter, use the name of the chlorophyll variable that was
    displayed above in dataInfo: **parameter \<- “chlorophyll”.** You
    can set this manually, but in this example, we will set
    **pamameter** directly from the variable returned from the
    rerddap::info() function (dataInfo).  
  - The metadata from dataInfo also shows you that this variable has an
    altitude coordinate that equals zero. Set the value of the z
    coordinate to zero: **zcoord \<- 0.**  
  - Obtain the beginning and ending dates from the variable returned
    from the rerddap::info() function (dataInfo).  
  - Use the “last” option for the ending date instead of the actual date
    in the dataInfo

<!-- end list -->

``` r
# Extract the parameter name from the metadata in dataInfo
parameter <- dataInfo$variable$variable_name

# Set the altitude coordinate to zero
zcoord <- 0.

# Extract the beginning and ending dates of the dataset from the metadata in dataInfo
global <- dataInfo$alldata$NC_GLOBAL
tt <- global[ global$attribute_name %in% c('time_coverage_end','time_coverage_start'), "value", ]

# Populate the time vector with the time_coverage_start from dataInfo
# Use the "last" option for the ending date
tcoord <- c(tt[2],"last")
```

\*\* Run the SeaWiFS data extraction with rxtracto\_3D:

``` r
# Extract the timeseries data using rxtracto_3D
chlSeaWiFS<-rxtracto_3D(dataInfo,
                        parameter=parameter,
                        tcoord=tcoord,
                        xcoord=xcoord,
                        ycoord=ycoord,
                        zcoord=zcoord)
```

    ## Registered S3 method overwritten by 'httr':
    ##   method           from  
    ##   print.cache_info hoardr

    ## info() output passed to x; setting base url to: https://coastwatch.pfeg.noaa.gov/erddap/

``` r
# Remove extraneous zcoord dimension for chlorophyll 
chlSeaWiFS$chlorophyll <- drop(chlSeaWiFS$chlorophyll)
```

### Get the MODIS data

  - First get the datadet metadata with “rerddap::info”
  - Use the MODIS dataset ID “erdMH1chlamday”

<!-- end list -->

``` r
# Use rerddap to get information about the dataset
# if you encouter an error reading the nc file clear the rerrdap cache: 
# rerddap::cache_delete_all(force = TRUE)
dataInfo <- rerddap::info('erdMH1chlamday')
dataInfo
```

    ## <ERDDAP info> erdMH1chlamday 
    ##  Base URL: https://upwell.pfeg.noaa.gov/erddap/ 
    ##  Dataset Type: griddap 
    ##  Dimensions (range):  
    ##      time: (2003-01-16T00:00:00Z, 2021-04-16T00:00:00Z) 
    ##      latitude: (-89.97917, 89.97916) 
    ##      longitude: (-179.9792, 179.9792) 
    ##  Variables:  
    ##      chlorophyll: 
    ##          Units: mg m-3

**Set the arguments for MODIS extraction**

Since this dataset does not have an altitude dimension, remove zcoord as
an argument in rxtracto\_3D

``` r
# Extract the parameter name from the metadata in dataInfo
parameter <- dataInfo$variable$variable_name

#Extract the start and end times of the dataset from the metadata in dataInfo
global <- dataInfo$alldata$NC_GLOBAL

# Populate the time vector with the time_coverage_start from dataInfo
# Use the "last" option for the ending date
tt <- global[ global$attribute_name %in% c('time_coverage_end','time_coverage_start'), "value", ]
tcoord <- c(tt[2],"last")
```

**Run MODIS extraction**

``` r
#Run rxtracto_3D
chlMODIS<-rxtracto_3D(dataInfo,
                      parameter=parameter,
                      tcoord=tcoord,
                      xcoord=xcoord,
                      ycoord=ycoord)
```

    ## info() output passed to x; setting base url to: https://upwell.pfeg.noaa.gov/erddap/

### Get VIIRS data

First get the dataset metadata with “rerddap::info” by changeing the
dataset ID to “nesdisVHNSQchlaMonthly”

Repeat the same commands but change the name of the dataset.

``` r
# Use rerddap to get information about the dataset
# if you encouter an error reading the nc file clear the rerrdap cache: 
# rerddap::cache_delete_all(force = TRUE)
#url <- 'https://coastwatch.pfeg.noaa.gov/erddap/'
#dataInfo <- rerddap::info('nesdisVHNSQchlaMonthly',url=url)

dataInfo <- rerddap::info('erdVHNchlamday')  # alternate dataset to use
dataInfo
```

    ## <ERDDAP info> erdVHNchlamday 
    ##  Base URL: https://upwell.pfeg.noaa.gov/erddap/ 
    ##  Dataset Type: griddap 
    ##  Dimensions (range):  
    ##      time: (2015-03-16T00:00:00Z, 2021-04-16T00:00:00Z) 
    ##      altitude: (0.0, 0.0) 
    ##      latitude: (-0.10875, 89.77125) 
    ##      longitude: (-180.03375, -110.00625) 
    ##  Variables:  
    ##      chla: 
    ##          Units: mg m^-3

**Set the arguments for VIIRS Extraction**

  - This dataset has an altitude dimension. Include zcoord as an
    argument in the rxtracto\_3D function

<!-- end list -->

``` r
zcoord <- 0.

## This extracts the parameter name from the metadata in dataInfo
parameter <- dataInfo$variable$variable_name

#Extract the start and end times of the dataset from the metadata in dataInfo
global <- dataInfo$alldata$NC_GLOBAL

# Populate the time vector with the time_coverage_start from dataInfo
# Use the "last" option for the ending date
tt <- global[ global$attribute_name %in% c('time_coverage_end','time_coverage_start'), "value", ]
tcoord <- c(tt[2],"last")
#tcoord <- c(tt[1], tt[2])
```

**Run VIIRS extraction**

``` r
# Run rxtracto_3D
chlVIIRS<-rxtracto_3D(dataInfo,
                      parameter=parameter,
                      tcoord=tcoord,
                      xcoord=xcoord,
                      ycoord=ycoord,
                      zcoord=zcoord)
```

    ## info() output passed to x; setting base url to: https://upwell.pfeg.noaa.gov/erddap/

``` r
## Remove extraneous zcoord dimension for chlorophyll 
chlVIIRS$chla <- drop(chlVIIRS$chla)
```

## Create timeseries of mean montly data

**The script below:**

  - spatially averages data for each time step within the area
    boundaries for each dataset.  
  - temporally averages data for data in each timeseries onto a map, for
    each dataset.

<!-- end list -->

``` r
## Spatially average all the data within the box for each dataset.
## The c(3) indicates the dimension to keep - in this case time 
chlSeaWiFS$avg <- apply(chlSeaWiFS$chlorophyll, c(3),function(x) mean(x,na.rm=TRUE))
chlMODIS$avg <- apply(chlMODIS$chlorophyll, c(3),function(x) mean(x,na.rm=TRUE))
chlVIIRS$avg <- apply(chlVIIRS$chla, c(3),function(x) mean(x,na.rm=TRUE))

## Temporally average all of the data into one map 
## The c(1,2) indicates the dimensions to keep - in this case latitude and longitude  
chlSeaWiFS$avgmap <- apply(chlSeaWiFS$chlorophyll,c(1,2),function(x) mean(x,na.rm=TRUE))
chlMODIS$avgmap <- apply(chlMODIS$chlorophyll,c(1,2),function(x) mean(x,na.rm=TRUE))
chlVIIRS$avgmap <- apply(chlVIIRS$chla,c(1,2),function(x) mean(x,na.rm=TRUE))
```

## Plot time series

  - Displays a timeseries plot of all three datasets

<!-- end list -->

``` r
## To print out a file uncomment the png command and the dev.off command
##png(file="CHL_timeseries.png", width=10,height=7.5,units="in",res=500)
plot(as.Date(chlSeaWiFS$time), chlSeaWiFS$avg, 
     type='b', bg="blue", pch=21, xlab="", cex=.7,
     xlim=as.Date(c("1997-01-01","2019-01-01")),
     ylim=c(0,3),
     ylab="Chlorophyll", main=ttext)

# Now add MODIS and VIIRS  data 
points(as.Date(chlMODIS$time), chlMODIS$avg, type='b', bg="red", pch=21,cex=.7)
points(as.Date(chlVIIRS$time), chlVIIRS$avg, type='b', bg="black", pch=21,cex=.7)

text(as.Date("1997-03-01"),2.8, "SeaWiFS",col="blue", pos=4)
text(as.Date("1997-03-01"),2.5, "MODIS",col="red", pos=4)
text(as.Date("1997-03-01"),2.2, "VIIRS",col="black", pos=4)
```

![](timeseries_satellite-data_files/figure-gfm/plot-1.png)<!-- -->

``` r
#dev.off() # This closes the png file if its been written to 
```

## Now add ESA OCCI Data

If you needed a single timeseries from 1997 to present, you would have
to use the plot above to devise some method to reconcile the difference
in values where two datasets overlap. Alternatively, you could use the
ESA OC-CCI (ocean color climate change initiative) dataset, which blends
data from many satellite missions into a single dataset. Next we will
add the ESA OC-CCI dataset to the plot above to see how it compares with
data from the individual satellite missions.

  - Change the dataset ID to “pmlEsaCCI31OceanColorMonthly” in the
    rerddap::info function.
  - There are over 60 variables in this dataset, so the dataInfo is not
    displayed (feel free to examine the dataInfo variable on your
    own).  
  - This dataset has no altitude dimension. Do not include zcoord as an
    argument in the rxtracto\_3D
function.

<!-- end list -->

``` r
# Reading in three datasets, which  have different datset attributes (ie parameter 
# name and the presence or absence of an altitude field) is cumbersome.  ESA makes 
# a "mission-less" product, which seemlessly integrates data from all these sensors 
# into one.  So lets redo this exercise iusing this dateset instead and compare the results.  

dataInfo <- rerddap::info('pmlEsaCCI50OceanColorMonthly', url=url)
                       
# This identifies the parameter to choose - there are > 60 in this dataset!
parameter <- 'chlor_a'

global <- dataInfo$alldata$NC_GLOBAL
tt <- global[ global$attribute_name %in% c('time_coverage_end','time_coverage_start'), "value", ]
tcoord <- c(tt[2],"last")
# if you encouter an error reading the nc file clear the rerrdap cache: 
rerddap::cache_delete_all(force = TRUE)

chlOCCCI<-rxtracto_3D(dataInfo,
                      parameter=parameter,
                      tcoord=tcoord,
                      xcoord=xcoord,
                      ycoord=ycoord)
```

    ## info() output passed to x; setting base url to: https://coastwatch.pfeg.noaa.gov/erddap/

``` r
# Now spatially average the data into a timeseries
chlOCCCI$avg <- apply(chlOCCCI$chlor_a, c(3),function(x) mean(x,na.rm=TRUE))

# Now temporally average the data into one map 
chlOCCCI$avgmap <- apply(chlOCCCI$chlor_a,c(1,2),function(x) mean(x,na.rm=TRUE))
```

**Add ESA OCCI data to the plot**

``` r
## Plot SeaWIFS
plot(as.Date(chlSeaWiFS$time), chlSeaWiFS$avg, 
     type='b', bg="blue", pch=21, xlab="", cex=.7,
     xlim=as.Date(c("1997-01-01","2019-01-01")),
     ylim=c(0,3),
     ylab="Chlorophyll", main=ttext)

## Add MODIS, VIIRS and OCCCI data 
points(as.Date(chlMODIS$time), chlMODIS$avg, type='b', bg="red", pch=21,cex=.7)
points(as.Date(chlVIIRS$time), chlVIIRS$avg, type='b', bg="black", pch=21,cex=.7)
points(as.Date(chlOCCCI$time), chlOCCCI$avg, type='b', bg="green", pch=21,cex=.5)
## Add text annotation for legend
text(as.Date("1997-03-01"),2.8, "SeaWiFS",col="blue", pos=4)
text(as.Date("1997-03-01"),2.5, "MODIS",col="red", pos=4)
text(as.Date("1997-03-01"),2.2, "VIIRS",col="black", pos=4)
text(as.Date("1997-03-01"),1.9, "OC-CCI",col="green", pos=4)
```

![](timeseries_satellite-data_files/figure-gfm/plotall-1.png)<!-- -->

``` r
#dev.off() # This closes the png file if its been written to 
```

## Make maps of the average chlorophyll for each satellite mission

The average chlorophyll was saved earlier as the chl avgmap variable

``` r
coast <- map_data("worldHires", ylim = ycoord, xlim = xcoord)

# Put arrays into format for ggplot
melt_map <- function(lon,lat,var) {
  dimnames(var) <- list(Longitude=lon, Latitude=lat)
  ret <- melt(var,value.name="Chl")
}

# Loop for making 4 maps
datasetnames <- c("SeaWiFS","MODIS","VIIRS","OC-CCI")

plot_list = list()

for(i in 1:4) {
  
  if(i == 1) chl <- chlSeaWiFS
  if(i == 2) chl <- chlMODIS
  if(i == 3) chl <- chlVIIRS
  if(i == 4) chl <- chlOCCCI
  
   chlmap <- melt_map(chl$longitude, chl$latitude, chl$avgmap)

   p = ggplot(
     data = chlmap, 
     aes(x = Longitude, y = Latitude, fill = log(Chl))) +
         geom_tile(na.rm=T) +
         geom_polygon(data = coast, aes(x=long, y = lat, group = group), fill = "grey80") +
         theme_bw(base_size = 12) + ylab("Latitude") + xlab("Longitude") +
         coord_fixed(1.3, xlim = c(-120,-116), ylim = ycoord) +
         scale_fill_gradientn(colours = rev(rainbow(12)), na.value = NA, limits=c(-2,3)) +
         ggtitle(paste("Average", datasetnames[i])
      ) 

  plot_list[[i]] = p
}

# Now print out maps into a png file.  Can't use par function with **ggplpot** to get 
# multiple plots per page.  Here using a function in the **gridExtra** package

#png(file="CHL_averagemaps.png")
library(grid)
grid.arrange(plot_list[[1]],plot_list[[2]],plot_list[[3]],plot_list[[4]], nrow = 2)
```

![](timeseries_satellite-data_files/figure-gfm/maps-1.png)<!-- -->

``` r
#dev.off()
```
