---
title: "3. Structure of sftrack/sftraj objects"
output:
  pdf_document: default
  html_document: default
vignette: |
  %\VignetteIndexEntry{3. Structure of sftrack/sftraj object}
   %\VignetteEncoding{UTF-8}
   %\VignetteEngine{knitr::rmarkdown}
---
```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
devtools::load_all("/home/matt/r_programs/sftrack")
#library(sftrack)

```
This is section will go more depth into the structure of sftrack/sftraj objects and how to work with them.

To begin, `sftrack` and `sftraj` objects are essentially data.frame objects with the 3 required columns (burst, geometry, and time). However, they are also of the subclass `sf`. This allows them to act like sf objects when working with functions in the `sf` package but act as `sftrack` objects for all other actions. It shold be noted that when possible sftrack objects will mimic sf functionality and thus many of the same words and tactics are used.

```{r}
# Make tracks from raw data
data <- read.csv(system.file('extdata/raccoon_data.csv', package='sftrack'))
data$month <- as.POSIXlt(data$acquisition_time)$mon+1

data$time <- as.POSIXct(data$acquisition_time, tz='EST')
coords = c('longitude','latitude')
burst = list(id = data$sensor_code, month = as.POSIXlt(data$acquisition_time)$mon+1)
time = 'time'
error = 'fix'
crs = '+init=epsg:4326'
my_sftrack <- as_sftrack(data = data, coords = coords, burst = burst, time = time, error = error, crs = crs)
my_sftraj <- as_sftraj(data = data, coords = coords, burst = burst, time = time, error = error, crs = crs)

```

When creating an object, `sftrack` creates the appropriate `sf` geometry column for the class, it then creates the burst class and bundles all the data and creates an `sf` object using sf::st_as_sf(). Finally, this input is used to create the `sftrack` object. Because of this there are 5 novel attributes in an `sftrack` object. Because it is also an `sf` object, the `sf_column` and `agr` attributes are created and used by sf. Addtionally, `burst`, `time`, and `error` are creatd by `sftrack` to describe which column contains the appropriate data set. `sftraj` is identical.

```{r}
attributes(my_sftrack)
```

The `sftrack` level attributes are simply pointers to the data. Any attributes relevant to the burst or geometry are stored in those columns themselves. 
### Geometry

The geometry column contains all attributes you would expect from an `sfc` object. We'll discuss in more detail in latre sections, but an `sftrack` object is a collection of `POINT`. NA points are allowed and converted to Empty points. 
```{r}
my_sftrack$geometry
```

An `sftraj` differs only in the geometry column. Where is it a `GEOMETRY` with mixture of `LINESTRINGS` **AND** `POINTS`. Since `LINESTRINGS` can not have one point of the line be NULL, when t1 is a known point but t2 is an Unknown point then the geometry row is a `POINT` object of t1. When t1 is unknown this row is an empty `POINT` object, regardless of the known status of t2. This allows lossless conversion between sftrack and sftraj while still having `sf` treat the data appropriately.


```{r}
  df1 <- data.frame(
    id = c(1, 1, 1, 1,1,1),
    month = c(1,1,1,1,1,1),
    x = c(27, 27, 27, NA,29,30),
    y = c(-80,-81,-82,NA, 83,83),
    timez = as.POSIXct('2020-01-01 12:00:00', tz = 'UTC') + 60*60*(1:6)
  )

  test_sftraj <- as_sftraj(data = df1,burst=list(id=df1$id, month = df1$month),
    time = df1$timez, active_burst = c('id','month'), coords = df1[,c('x','y')])
  test_sftraj$geometry
```

##### Burst
The burst column is very specialized, and we will cover it in it's own section. At present, the only special attributes it stores is the 'active_burst'. A burst column is a class `multi_burst` consisuting of multiple `ind_burst`s. All multi_bursts have 2 criteria: They contain the same active_bursts and the same grouping names. 

```{r}
attributes(my_sftrack$burst)
summary(my_sftrack)
```

The time and error columns are not particularly specialized at this moment. The time column is either an integer or POSIXct relative to the users input, but the entire column must remain that type.

The error column is the column with the relevant error information for the spatial points in it. At present we have not built particular functionality but plan to in the future or reserve this for other developers to build upon.


## Subsetting

An sftrack object attempts to act like a data.frame and sf whenever appropriate. Because of this you can subset an sftrack object as you would a data.frame. Except, like sf, it attempts to retain the geometry, burst, and time columns, in order to maintain sftrack status.

In this way row subsetting is very straight forward, as each row represents an individual point in time.

```{r}

my_sftrack[1:10,]
```

Subsetting by column, however, sftrack attempts to retain important columns by default. This mirrors sf functionality except with multiple columns

```{r}

my_sftrack[1:3,c(1:3)]
```
}

To turn off this feature, you just use the drop = T argument which returns a data.frame object instead, largely mimicing the use in `sf`.

```{r}
my_sftrack[1:3,c(1:3), drop = TRUE]
```

`sftraj`s work nearly the same as `sftrack`s, however because they are a step model where the steps are modeled as step1 (t1 ->t2) its important to note that subsetting will not automatically recalculate any new steps for you even if the original t2 point has been deleted.

If your subsetting will also change the end points for steps, then you can recalculate using `step_recalc()`. The output which is your original straj object but with the geometry column recalculated to the new t2s. 

```{r}
plot(my_sftraj, main ='original', axes = T)
new_traj <- my_sftraj[seq(10,nrow(my_sftraj),10),]

plot(new_traj, main ='before recalculation', axes = T)

plot(step_recalc(new_traj),  main ='after recalculation', axes = T)
```
### Some basic functionality of sf_track and sf_traj objects

##### print  

`print()` prints out the type of object as well as specific data on the sf_track object. Additionally you can supply the number of rows or columns you'd like to display with arguments `n_row` and `n_col`. When using `n_col` the display will show the `burst` `geometery`, and `time` fields as well as any other columns starting from column 1 until `#columns + 3 = n_col`. If neither is provided then print just uses default values in the global options. `n_col` and `n_row` are optional arguments, defaults to data.frame defaults.

```{r}

print(my_sftrack,5,10)
```

##### summary  

`summary()` works as youd expect for a data.frame, except it displays the burst column as a count of each active_burst combination and the `active_burst `for that column.  
```{r}
summary(my_sftrack)
```

##### summary_sftrack  

`summary_sftrack()` is a special summary function specific for sftrack objects. It summarizes the data based on the beginning and end of each burst as well as the total distance of the burst. This function uses `st_distance` from the `sf` package and therefore outputs in units of the crs. In this example the distance is in meters.
```{r}
summary_sftrack(my_sftrack)

```

You can also trigger this function by using `summary(data, stats = TRUE)`

```{r}
summary(my_sftrack, stats = TRUE)
```


