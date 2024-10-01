# Maxent demonstration using R

In this exercise, you will fit a species distribution model (SDM) for a threatened bumblebee species (Mosshumla, Bombus muscorum) in Sweden. The model will be fitted using the Maxent algorithm, which is a popular machine learning algorithm for SDMs. The model will be trained using bioclimatic variables from the WorldClim dataset and occurrence data for the species sourced from Artportalen (https://artportalen.se/). The model will be used to predict the potential distribution of the species throughout southern Sweden.

## SETUP R PROJECT

- Set work directory

The next step is to make sure you are working in the correct directory. This is an important step because we want to make sure our data and outputs are saved clearly inside our project folder. So, we first make sure that we are working inside our project directory. To check the current project directory use getwd() function. And to change the current working directory to a new one, use the setwd() function

``` r
# Check current working directory
getwd()
```

Download the mosshumla occurrence dataset from the data folder (https://github.com/liamkendall/sdm-exercise/tree/main/data) and put it in a new folder called data in your working directory.


## STEP 1: Install required libraries

For this exercise, we need to install some R packages. Make sure to run them
for first time.

Dismo, SDMtune are for fitting SDMs.
Terra and sf are for processing spatial data.
Viridis is for color palettes.

``` r
install.packages("dismo",dependencies = T)
install.packages("SDMtune",dependencies = T)
install.packages("terra",dependencies = T)
install.packages("viridis",dependencies = T)
install.packages("sf",dependencies = T)
```

After installing the packages, we need to load them

``` r
library(SDMtune)
library(dismo)
library(terra)
library(geodata)
library(viridis)
library(sf)
```

## Step 2 : Prepare data for model

There are different ways of implementing a SDM. Here we will use a common workflow for fitting Maxent SDMs. The workflow involves the following steps:
1) Defining the study area/boundaries for model fitting and spatial predictions
2) Downloading environmental data (bioclimatic variables) for the study area
3) Obtaining species occurrence data for the target species
4) Preparing the data for model fitting
5) Training the Maxent model
6) Assessing model performance
7) Visualizing the model results

## 2.1 Get a shapefile of (Southern) Sweden

First, we create a boundary box limited by the range of species to be modelled. We will use this boundary box to crop the shapefile and raster data to the study area extent. I have already set this based upon species occurrences.

``` r
ext = c(11.0273686052, 20, 23.9033785336, 62) 
plot(ext(ext))
```

Second, we will download the shapefile for Sweden from the GADM database. The GADM database provides administrative boundaries for countries at different levels of detail. We will download the shapefile for Sweden at the first administrative level (län) using the `gadm()` function from the `geodata` package.

``` r
#help(getData,raster)
bd  = gadm(country='Sweden',level=1,path = "./data",download = T)
bd_main = bd # saving in a new variable to use later
```

Third, we crop the country shape file to study area extent

``` r
bd  = crop(bd,extent(ext))
```

Finally, we plot the shapefile to visualize the study area boundary. We can also overlay the cropped shapefile on the original shapefile to see the difference.

``` r
par(mfrow=c(1,2))
plot(bd_main,axes=T, main="Sweden län")
plot(bd,axes=T, main="Cropped shapefile")
```


## 2.2 Raster/Image data: Climatic data

The worldclim dataset provides global bioclimatic variables at different spatial resolutions. We will download 19 bioclimatic variables for Sweden at a resolution of 0.5 degrees (~1km). The bioclimatic variables are derived from monthly temperature (variables 1:11) and precipitation (12:19) data and are commonly used as environmental predictors in SDMs. See the WorldClim website for more information on the 19 bioclimatic variables and data sources.

``` r
bio19 = geodata::worldclim_country(
  country='Sweden',
  var='bio',
  res=0.5,
  path = "./data",
  download=T
  )

```

### Plotting raster images

We can check the images using default plot function.

``` r
 # crop the data to the extent of the study area
bio19 = crop(bio19,extent(ext)) 

# HOW TO PLOT RASTER AND SHAPEFILE
plot(bio19[[1]],main="Annual mean temperature",col=map.pal("viridis", 100))
plot(bd,add=T)
```


Separately we can plot first four layers also.

``` r
# first four temperature layers
plot(bio19[[1:4]],col=map.pal("viridis", 100))

# first four rainfall layers
plot(bio19[[12:16]],col=map.pal("viridis", 100))

# saving the results
pdf("./result/bio1-4.pdf",width = 6,height = 4)
plot(bio19[[1:4]],col=map.pal("viridis", 100))
dev.off()

pdf("./result/bio12-16.pdf",width = 6,height = 4)
plot(bio19[[12:16]],col=map.pal("viridis", 100))
dev.off()
```

## STEP 4: Prepare data for Maxent

For maxent SDM we need two types of data: species occurrence data and background data. Species occurrence data are the locations where the species has been observed, while background data are randomly sampled points across the study area (i.e., pseudo-absences). The Maxent algorithm uses these two types of data to model the relationship between the species and the environmental variables.

Refer to the Maxent documentation for more information on the Maxent algorithm and its implementation.
Phillips, S. J. (2005). A brief tutorial on Maxent. AT&T Research, 190(4), 231-259.


###  Presence points

``` r
# load species occurrence data 
occ = read.csv("./data/mosshumla.csv",sep=";")

#these functions convert the data to spatial data frame
occ=st_as_sf(occ,coords=c("Ost","Nord"),crs=3021) 
#and then transform the data to the appropriate projection (WGS84)
occ = st_transform(occ,4326)

# extract just the coordinates of each occurrence record
occ_coordinates=st_coordinates(occ)
```

We randomly sample 1000 points across our study area to represent background points, to generate 'pseudo-absences'.

``` r
# Absense points
bg = predicts::backgroundSample(n=1000,
                                bio19[[1]])
```


### Using SDMtune to fit the model

We then use the SDMtune to set up the data for model training. This first step creates an SWD (sample with data) object for use in model training. The SWD object contains the species occurrence data, background data, and environmental data.

``` r
# Prepare data SWD object which actually only requires for SDMtune model
# If we want to use other package we do not need this format. But, SDMtune
# package is very useful, later we will discover.
data <- SDMtune::prepareSWD(
  species = "Mosshumla", #species name
  p = occ_coordinates, #coordinates
  a = bg, #back ground points
  env = bio19, #environmental data
  )
```

You can check the SWD object using @ (at) sign to see how the data is stored

``` r
# check the data object, what is inside
data@species
data@data[1:10,1:3]
data@pa[1:6]
data@coords[1:6,]
```


## Step 5. Train a simple maxent model

We can then use the train() function from the SDMtune package to train a Maxent model. The train() function takes several arguments, including the method (in this case, "Maxnet" i.e., Maxent in R), the feature classes to use (e.g., linear, quadratic, hinge), the regularization value, and the number of iterations. The train() function returns a trained model object that can be used to make predictions on new data.

See https://onlinelibrary.wiley.com/doi/10.1111/j.1472-4642.2010.00725.x for more details on feature classes and regularization values.


``` r
mx1 = train(
  method = "Maxnet",  # the name of the model
  fc="lqh",           # feature classes: linear, quadratic, hinge
  reg=0.5,            # regularization parameters
  verbose = T,        # show message during training
  data = data,        # the data variable
  iter = 1000         # number of iterations the logarithm will run
  )    
```


### Model results
SDMtune has several functions to show the model results. First we can plot the response curves against mean annual temperature (bio_1) and mean annual precipitation (bio_12)

``` r
# plot the response curve
plotResponse(mx1, var = c("wc2.1_30s_bio_1"), type = "logistic") #mean annual temperature
plotResponse(mx1, var = c("wc2.1_30s_bio_12"), type = "logistic") #mean annual precipitation
```

Then we can plot the variable importance using AUC (area under curve). This gives us an idea of how well the model predicts the species distribution based on the environmental variables. Closer to 1 means more accurate predictions.

``` r  
# Then we plot the ROC curve for our model which shows the traing AUC
auc(mx1)        # just to get the model AUC
plotROC(mx1)    # to plot the AUC curve
```

### Apply model prediction

We can then predict the probability of occurrence in each pixel/grid cell across our study area

``` r
map = predict(mx1, data=bio19, type="logistic")
```

## Step 6. Visualize results

Finally, we use the plotPred function to generate a map of the predicted probability of occurrence across our study area. The plotPred function takes several arguments, including the map object (i.e., the predicted probability of occurrence), the legend title, and the color ramp to use for the map.

``` r
dev.off() # clear the plotting window first
# saving the results
pdf("./result/predictions.pdf",width = 6,height = 4)
par(pty="s")
plotPred(map, 
 lt = "Climate\nsuitability",
 colorramp = c("#2c7bb6", "#abd9e9", "#ffffbf", "#fdae61", "#d7191c"),hr=T
)
dev.off()
```

### Questions

1. How well does the model predict the occurrence of Bombus muscorum? refer to the AUC curve and the response curves for the two environmental variables. 
2. Compare the maps and response curves of MAT and MAP with the mapped predictions, what does it tell us about this species environmental tolerances in terms of temperature and rainfall?
3. Do you think climate change or anthropogenic land use change is more important for the distribution of Bombus muscorum in Sweden? Why?

### Fun

If there is time, navigate to Artportalen, download occurrence data for another bumblebee species, and fit a Maxent model for that species. Be careful to re-assess the spatial boundaries for the new species. Compare the results with the model for Bombus muscorum. What are the differences in the environmental variables that are important for the two species? What are the differences in the predicted distributions of the two species?


