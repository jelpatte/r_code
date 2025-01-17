# Define a marine habitat

> notebook filename | 07\_turtlewatch\_xtracto.Rmd

The TurleWatch project investigated the thermal habitat of loggerhead
sea turtles in the Pacific Ocean north of the Hawaiian Islands. Research
results indicate that most loggerhead turtles stay in water between
17.5°C and 18.5°C. When the 17.5°C to 18.5°C temperature contour is
drawn on a map of sea surface temperature conditions, it delineates the
boundary of the loggerhead’s preferred habitat.

In this exercise you will plot the thermal boundary of loggerhead sea
turtles using satellite sea surface temperature. The exercise
demonstrates the following techniques:  
\* Using xtracto\_3D to extract data from a rectangular area  
\* Masking a data array  
\* Plotting maps using
ggplot

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

list.of.packages <- c( "ncdf4", "rerddap", "plotdap", "RCurl",  
                       "raster", "colorRamps", "maps", "mapdata",
                       "ggplot2", "RColorBrewer", "rerddapXtracto")

# create list of installed packages
pkges = installed.packages()[,"Package"]

for (pk in list.of.packages) {
  pkgTest(pk)
}
```

## Select the Satellite Data

  - Use the MUR SST dataset (ID jplMURSST41mday)  
  - Gather information about the dataset (metadata) using **rerddap**  
  - Displays the information

<!-- end list -->

``` r
# CHOOSE DATASET and get information about it 


url = "http://coastwatch.pfeg.noaa.gov/erddap/"
dataInfo <- rerddap::info('jplMURSST41mday',url=url)
parameter <- 'sst'
```

## Get Satellite Data

  - Select an area off the coast of California: longitude range of -130
    to -115 east and latitude range of 25 to 40 north  
  - Set the time range to days withing one month:
    tcoord=c(‘2018-06-06’,‘2018-06-08’)). The values do have to be
    different.

<!-- end list -->

``` r
# latitude and longitude of the vertices
ylim<-c(25,40)
xlim<-c(-130,-115)

# Choose an area off the coast of California
# Extract the data
SST <- rxtracto_3D(dataInfo,xcoord=xlim,ycoord=ylim,parameter=parameter, 
                   tcoord=c('2018-06-06','2018-06-08'))
```

    ## Registered S3 method overwritten by 'httr':
    ##   method           from  
    ##   print.cache_info hoardr

    ## info() output passed to x; setting base url to: http://coastwatch.pfeg.noaa.gov/erddap/

``` r
# Drop command needed to reduce SST from a 3D variable to a 2D  one  
SST$sst <- drop(SST$sst) 
```

## Make a quick plot using plotBBox

``` r
plotBBox(SST, plotColor = 'thermal',maxpixels=100000)
```

    ## grid object contains more than 1e+05 pixels

    ## increase `maxpixels` for a finer resolution

    ## Warning in raster::projectRaster(r, crs = crs_string): input and ouput crs are
    ## the same

![](define_marine_habitat_files/figure-gfm/qplot-1.png)<!-- -->

## Define the Thermal niche of Loggerhead Turtles

\_\_ Set the thermal range to 17.5-18.5 degrees C, as determined by the
TurtleWatch program.\_\_

``` r
## Define turtle temperature range
min.temp <- 17.5
max.temp <- 18.5
```

**Create another variable for habitat temperature**

Set the habitat temperature to equal NA

``` r
SST2 <- SST
SST2$sst[SST2$sst >= min.temp & SST2$sst <= max.temp] <- NA
plotBBox(SST2, plotColor = 'thermal',maxpixels=100000)
```

    ## grid object contains more than 1e+05 pixels

    ## increase `maxpixels` for a finer resolution

    ## Warning in raster::projectRaster(r, crs = crs_string): input and ouput crs are
    ## the same

![](define_marine_habitat_files/figure-gfm/makeVar-1.png)<!-- -->

It would be nicer to color in the turtle habitat area (the NA values)
with a different color. If you want to customize graphs its better to
use `ggplot` than the `plotBBox` that comes with `rerrdapXtracto`
package. Here we will use `ggplot` to plot the data. But first the data
is reformatted for use in `ggplot`.

Restructure the data

``` r
dims <- dim(SST2$sst)
SST2.lf <- expand.grid(x=SST$longitude,y=SST$latitude)
SST2.lf$sst<-array(SST2$sst,dims[1]*dims[2])
```

## Plot the Data using ‘ggplot’

``` r
coast <- map_data("worldHires", ylim = ylim, xlim = xlim)

par(mar=c(3,3,.5,.5), las=1, font.axis=10)

myplot<-ggplot(data = SST2.lf, aes(x = x, y = y, fill = sst)) +
  geom_tile(na.rm=T) +
  geom_polygon(data = coast, aes(x=long, y = lat, group = group), fill = "grey80") +
  theme_bw(base_size = 15) + ylab("Latitude") + xlab("Longitude") +
  coord_fixed(1.3,xlim = xlim, ylim = ylim) +
  ggtitle(unique(as.Date(SST2$time))) +
  scale_fill_gradientn(colours = rev(rainbow(12)),limits=c(10,22),na.value = "firebrick4") 

myplot
```

![](define_marine_habitat_files/figure-gfm/plot-1.png)<!-- -->
