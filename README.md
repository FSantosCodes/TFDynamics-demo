# TFDynamics Demo 

Fabian Santos Ph.d.

23 August 2017  

This is a R package for process Landsat 4-8 images and perform multi-date classification and change detection. For details of its technical approach you can visit:

 * Santos F., Meneses P. & Hostert P., 2018. Monitoring Long-Term Forest Dynamics with Scarce Data: A Multi-Date Classification Implementation in the Ecuadorian Amazon.     
  European Journal of Remote Sensing. DOI: 10.1080/22797254.2018.1533793

While the code is funcional, the project was discontinued as Google Earth Engine  emerged as the prime platform for massive processing of spatial data. While this R implementation is demostrated in the publication, it is time demanding due landsat imagery download and processing. Neverthless, the preprocessing approach of this work is now implemented in GEE (check: https://code.earthengine.google.com/029d005e4a4d7e48b295fe968816b3a1)

The next lines of code shows a demo of its R implementation.

### (1) Required R libraries

```r
#simple function for install or load libraries
get.libraries <- function(libraries){
  for (i in libraries){
    if(!require(i,character.only=T)){
      install.packages(i)
      library(i,character.only=T)}
  }
}
libraries <- c("caret","Cairo","caTools","corrplot","doParallel","foreach","gdalUtils","ggplot2","gtools","imager",
               "maptools","matrixStats","mmand","plyr","proj4","randomForest","raster","RColorBrewer","cowsay",
               "RcppRoll","reshape2","rgdal","rgeos","RStoolbox","samplingbook","scales","tidyr","zoo")
get.libraries(libraries)
#load TFDynamics
library(TFDynamics)
```
### (2) Start unzipping images downloaded from Earth Explorer (Level 1T; see:https://espa.cr.usgs.gov/)

```r
#first assign a working folder and print some information about zip files
base.folder <- "H:/Fabian/Geoinformation/images_Esmeraldas"
head(list.files(base.folder))
```

```
## [1] "ANALYSIS"    "IMAGES"      "OTHERS"      "SCRIPTS_LOG" "ZIP"
```

```r
paste0(length(list.files(base.folder))," images zip files...")
```

```
## [1] "5 images zip files..."
```

```r
#As TFDynamics works in parallel, let´s see how many cores we had in this computer
detectCores()
```

```
## [1] 8
```

```r
#Start unzipping function. Beware using all cores as computer can be slowed drastically and RAM memory overloaded
#images_unzip(zip_folder=images.esmeraldas,number_cores=8)
#after function is over, TFDynamics organize images files in different folders and subfolders
list.files(base.folder)
```

```
## [1] "ANALYSIS"    "IMAGES"      "OTHERS"      "SCRIPTS_LOG" "ZIP"
```

```r
#where /IMAGES folder contains unzipped images and each folder correspond to an image and its acquisition date
head(list.files(paste0(base.folder,"/IMAGES")))
```

```
## [1] "1985-04-21" "1986-02-19" "1986-05-10" "1986-11-02" "1987-02-06"
## [6] "1987-09-02"
```

```r
#Each folder-date had an structure like this:
list.files(list.files(paste0(base.folder,"/IMAGES"),full.names=T)[1])
```

```
## [1] "ETC"     "INDICES" "MASK"    "META"    "RAD_COR" "RGB"
```

```r
#where /RAD_COR is used to store images bands products, /META for metadata and /ETC for all other not relevant files. Additional folders are progresively created by TFDynamics for specific functions explained later in this demo.
```
### (3) Inspect images files

```r
#first we will validate if all images files are complete and can be processed by TFDynamics, therefore we applied the next function 
images_validation(base.folder)
```

```
## [1] "Metadata files validated!"
## [1] "Path & Row validated!"
## [1] "Projection validated!"
## [1] "Bands files validated!"
## [1] "Mask files validated!"
## [1] "Average extent shapefile created!"
## [1] "*** Script succesfully finished ***"
```

```
## [1] "Execution.time 7.85 seconds"
```

```r
#if an error pops up, it will indicate the image folder-date that should be check. When finish, images bands files are further organized and a new folder is created which contains a shapefile:
list.files(paste0(base.folder,"/ANALYSIS/MASK"),pattern=".shp")
```

```
## [1] "average_extent_area.shp"
```

```r
#this is just in case different extents are observed (happens with complete images) and this shapefile correspond to the maximun extent observed in collection
```
### (3) Clip & mask images

```r
#after images files are validated, now we are going to clip them with an study area shapefile. This could be avoid if images are already clipped (recommended to do at ESPA) but for ilustration we apply the function:
images_clip(base_folder=base.folder,
            subfolder_prefix="IMAGES", #'IMAGES' specify all files found in this subfolder but can be subfolder specific
            clip_filename="H:/Fabian/Geoinformation/images_Esmeraldas/OTHERS/clip_shp/study_area.shp",
            number_cores=8,
            temp_folder="H:/temp") #this folder avoids default storage of temporals in root drive and flood it with trash
#then cloud and water masks (or LEDAPS, CFmask) are merged into one called no data mask. For this we apply the function:
images_mask(base_folder=base.folder,
             number_cores=8,
             temp_folder="H:/temp")
```
### (4) Images selection

```r
#Now we tabulate images metadata files for selected best images. For this we apply:
landsat.data <- metadata_tabulation(base_folder=base.folder,
                                    output_prefix="initial",
                                    scale_plot=2,
                                    number_cores=8,
                                    return_outputs=T)
#then we can observe a plot reporting nodata mask cover in the collection (jpg plot is save at /ANALYSIS/METADATA)
landsat.data[[2]]
#with tabulation is possible to obtain the number of images >90% nodata and without orthorectification
table(as.numeric(landsat.data[[1]]$nodata_msk)>90)
table(is.na(landsat.data[[1]]$geometric_RMSE))
#and since their quality is bad we can move them applying:
move.img <- landsat.data[[1]][as.numeric(landsat.data[[1]]$nodata_msk)>=90|is.na(landsat.data[[1]]$geometric_RMSE),] 
move.img <- move.img$folder
images_move(base_folder=base.folder,
            search_pattern=move.img,
            output_folder="H:/Fabian/Geoinformation/images_Esmeraldas/OTHERS/bad_quality", #this folder should be created before
            only_files=F)
#and finally we tabulate again the depured collection and plot its report
landsat.data <- metadata_tabulation(base_folder=base.folder,
                                    output_prefix="depured",
                                    scale_plot=2,
                                    number_cores=8,
                                    return_outputs=T)
landsat.data[[2]]
```
### (5) Derivating masks sums, RGB and indices

```r
#this chapter applies function for derive additional information for collection. Then we first make a sum of no data and water masks
#images_maskSum(base_folder=base.folder,
#               number_cores=8)
#if plotted, features such as permant water bodies or cloudy areas are reveal in the study area
plot(stack(list.files(paste0(base.folder,"/ANALYSIS/MASK"),pattern=".tif$",full.names=T)),col=bpy.colors(100))
```

![](TFDynamics_demo_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

```r
#now we are going to stack RGB from collection. RGBs´ can be visuallized on-screen from its quick version at /ANALYSIS/QUICK_VIEW
#images_RGB(base_folder=base.folder,
#           input_bands=c(7,4,3),
#           zoom_point=c(804736,136412),
#           background_color="darkgoldenrod1",
#           number_cores=8,
#           temp_folder="H:/temp")
plotRGB(stack(list.files(paste0(list.files(paste0(base.folder,"/IMAGES"),full.names=T)[1],"/RGB"),pattern=".tif$",full.names=T)))
```

![](TFDynamics_demo_files/figure-html/unnamed-chunk-6-2.png)<!-- -->

```r
#finally vegetation indices and other Landsat derivatives are calculated applying:
#images_indices(base_folder=base.folder,
#               RAD_COR_folder="SR",
#               indices_names=c("NDVI","GEMI","MSAVI","NBR","AFRI16","AFRI21","NDBI",
#                               "TCTB","TCTG","TCTW",
#                               "R34","R43","R23","R32","R45","R54","R57","R35","R72"),
#               number_cores=8,
#               temp_folder="H:/temp")
```
### (6) Compositing images

```r
#with all indices created, in this step they are composited using a median-based function. For the moment TFDynamics do not calculates percentiles but further development will consider it
composites_median(base_folder=base.folder,
                 time_step=36,
                 target_indices=c("NDVI","GEMI","MSAVI","NBR","AFRI16","AFRI21","NDBI",
                                 "TCTB","TCTG","TCTW",
                                 "R34","R43","R23","R32","R45","R54","R57","R35","R72"),
                 target_bands=c(1,2,3,4,5,7),
                 number_cores=6,
                 temp_folder="H:/temp")
#and for visualize resulting composites RGB are constructed from their outputs. In this ase we use 7,4,3  bands
composites_RGB(base_folder=base.folder,
               RGB_bands=c("band7", "band4", "band3"),
               number_cores=6,
               temp_folder="H:/temp")
```
### (7) Metrics calculation

```r
#for improve classification, we first derive additional metrics from SRTM downloaded from CGIAR (http://srtm.csi.cgiar.org/). As TFDynamics automatically adjust it, there is not need to reproject or crop to the study area
metrics_DEM(base_folder=base.folder,
            DEM_filename="H:/Fabian/Geoinformation/OTHER_DATA/Altitude_data/CGIAR/ecuador_dem_GWS.tif",
            reference_filename=NA)
```
### (8) Training classificator

```r
#for train a clasificator first we extract data from composites & DEM metrics
train.samples <- sampling_extraction(base_folder=base.folder,
                    date_composite="2002-04-12",
                    additional_metrics='DEM',
                    samples_shapefile="H:/Fabian/Geoinformation/images_Esmeraldas/OTHERS/training_samples/training_2002.shp",
                    classes_column="code",
                    scale_plot=2,
                    number_cores=7,
                    temp_folder=NA,
                    return_outputs = T)
#we plot its result to see classes spectral signatures
plot(load.image(list.files(paste0(base.folder,"/ANALYSIS/SAMPLING/DATA_EXTRACTION"),full.names=T)[2]),axes=FALSE)
#and with extrated data we now train a random forest classifier. Since TFDynamics used caret package, any classificator listed at: (https://topepo.github.io/caret/available-models.html) can be used but its dependencies should be installed before.
model.rf <- classification_training(base_folder=base.folder,
                                    data_extraction_filename=train.samples[[1]],
                                    algorithm_name="rf",
                                    iteration_number = 100,
                                    number_cores=8,
                                    return_outputs=T)
print(model.rf)
```
### (9) Predict and harmonize results

```r
#with the model we apply to composites for its classification. As it is saved in envi format we have to assign color for the output classes in maps. Additionaly, a median filter can be applied to improve results
#classification_prediction(base_folder=base.folder,
#                          model_name=model.rf,
#                          median_filter=3,
#                          color_classes=c("darkgreen","mediumorchid4","yellow","deeppink3","green"),
#                          overlap_mask=F,
#                          number_cores=8,
#                          temp_folder="H:/temp")
#after classification, we harmonize results for improve its quality. For this we asign a set of rules, which identify temporal patterns (which are ilogical) and replace them with new values. TFDynamics require for this a data frame indicating these rules:
pattern.class <- c("1,2,1","1,3,1","1,4,1","1,5,1",
                   "2,1,2","2,3,2","2,4,2","2,5,2",
                   "3,1,3","3,2,3","3,4,3","3,5,3",
                   "4,1,4","4,2,4","4,3,4","4,5,4",
                   "5,1,5","5,2,5","5,3,5","5,4,5")
replace.class <- c("1,1,1","1,1,1","1,1,1","1,1,1",
                   "2,2,2","2,2,2","2,2,2","2,2,2",
                   "3,3,3","3,3,3","3,3,3","3,3,3",
                   "4,4,4","4,4,4","4,4,4","4,4,4",
                   "5,5,5","5,5,5","5,5,5","5,5,5")
reclass.rules <- data.frame(pnt=pattern.class,rec=replace.class)
#then we apply harmonization over classifications
#classification_harmonize(base_folder=base.folder,
#                         pattern_recode=reclass.rules,
#                         NA_fill="mode",
#                         median_filter=3,
#                         number_cores=7,
#                         temp_folder="H:/temp")
#and plot an example of them
plot(raster(list.files(paste0(base.folder,"/ANALYSIS/CLASSIFICATION/FILTERED/"),full.names=T,pattern=".envi$")[1]))
```

![](TFDynamics_demo_files/figure-html/unnamed-chunk-10-1.png)<!-- -->![](TFDynamics_demo_files/figure-html/unnamed-chunk-10-2.png)<!-- -->
### (10) Synthesis map

```r
#finally, land cover maps are synthetized by the next function in order to extract vegetation-loss and regrowth in the time-period analyzed. Additionally, this function calculates land cover classes frequencies.
#classification_synthesis(base_folder=base.folder,
#                         vegetation_classes=c(1,2),
#                         regrowth_observations=2,
#                         number_cores=8,
#                         temp_folder="H:/temp")
plot(raster(list.files(paste0(base.folder,"/ANALYSIS/CLASSIFICATION/SYNTHESIS/"),full.names=T,pattern=".envi$")[1]))
```

![](TFDynamics_demo_files/figure-html/unnamed-chunk-11-1.png)<!-- -->![](TFDynamics_demo_files/figure-html/unnamed-chunk-11-2.png)<!-- -->

```r
plot(raster(list.files(paste0(base.folder,"/ANALYSIS/CLASSIFICATION/SYNTHESIS/"),full.names=T,pattern=".envi$")[2]))
```

![](TFDynamics_demo_files/figure-html/unnamed-chunk-11-3.png)<!-- -->![](TFDynamics_demo_files/figure-html/unnamed-chunk-11-4.png)<!-- -->






