Exploring maps
========================================================



```r
library(stringr)
library(ggplot2)
library(dplyr)
library(lubridate)
library(ggvis)
library(maps)
library(ggmap)
library(rworldmap)
library(grid)
library(scales)
```



```r
setwd("..")
sets <- c("item", "parent", "school", "scoredItem", "student")

# function to build the file names
fn_build <- function(file_name) {
    
    template <- c("2012.rda", "2012dict.rda")
    
    file_name %>% vapply(str_join, template, template) %>% file.path(".", "data", 
        .)
}

# load the data
sets %>% fn_build %>% lapply(load, .GlobalEnv)
```

```
## [[1]]
## [1] "item2012"
## 
## [[2]]
## [1] "item2012dict"
## 
## [[3]]
## [1] "parent2012"
## 
## [[4]]
## [1] "parent2012dict"
## 
## [[5]]
## [1] "school2012"
## 
## [[6]]
## [1] "school2012dict"
## 
## [[7]]
## [1] "scoredItem2012"
## 
## [[8]]
## [1] "scoredItem2012dict"
## 
## [[9]]
## [1] "student2012"
## 
## [[10]]
## [1] "student2012dict"
```

```r

# clean rm(fn_build, sets)
```



```r
# function to convert to data-frames
fn_make_df <- function(named_vector) {
    data.frame(variable = attr(named_vector, "names"), description = named_vector, 
        row.names = NULL)
}

# there's a clever way to do this, but beyond me for naw
dict_item2012 <- fn_make_df(item2012dict)
dict_parent2012 <- fn_make_df(parent2012dict)
dict_school2012 <- fn_make_df(school2012dict)
dict_scoredItem2012 <- fn_make_df(scoredItem2012dict)
dict_student2012 <- fn_make_df(student2012dict)

# clean rm(fn_make_df) rm(item2012dict, parent2012dict, school2012dict,
# scoredItem2012dict, student2012dict)
```



```r
extractPolygons <- function(shapes) {
    
    dframe <- ldply(1:length(shapes@polygons), function(i) {
        ob <- shapes@polygons[[i]]@Polygons
        dframe <- ldply(1:length(ob), function(j) {
            x <- ob[[j]]
            co <- x@coords
            data.frame(co, order = 1:nrow(co), group = j)
        })
        dframe$region <- i
        dframe$name <- shapes@polygons[[i]]@ID
        dframe
    })
    # construct a group variable from both group and polygon:
    dframe$group <- interaction(dframe$region, dframe$group)
    
    dframe
}

# To get a blank background on map
new_theme_empty <- theme_bw()
new_theme_empty$line <- element_blank()
new_theme_empty$rect <- element_blank()
new_theme_empty$strip.text <- element_blank()
new_theme_empty$axis.text <- element_blank()
new_theme_empty$plot.title <- element_blank()
new_theme_empty$axis.title <- element_blank()
new_theme_empty$plot.margin <- structure(c(0, 0, -1, -1), unit = "lines", valid.unit = 3L, 
    class = "unit")
```



```r
# Extract map polygons for modern world
world <- getMap(resolution = "low")
library(plyr)
```

```
## -------------------------------------------------------------------------
## You have loaded plyr after dplyr - this is likely to cause problems.
## If you need functions from both plyr and dplyr, please load plyr first, then dplyr:
## library(plyr); library(dplyr)
## -------------------------------------------------------------------------
## 
## Attaching package: 'plyr'
## 
## The following object is masked from 'package:lubridate':
## 
##     here
## 
## The following objects are masked from 'package:dplyr':
## 
##     arrange, desc, failwith, id, mutate, summarise, summarize
```

```r
world.polys <- extractPolygons(world)
detach("package:plyr")
# Subset data
student2012.sub <- student2012[, c(1:7, 500:550)]
colnames(student2012.sub)[1] <- "name"
student2012.sub$name <- as.character(student2012.sub$name)
# Check mismatches of names unique(anti_join(student2012.sub,
# world.polys)[1])
student2012.sub$name[student2012.sub$name == "Serbia"] <- "Republic of Serbia"
student2012.sub$name[student2012.sub$name == "Korea"] <- "South Korea"
student2012.sub$name[student2012.sub$name == "Chinese Taipei"] <- "Taiwan"
student2012.sub$name[student2012.sub$name == "Slovak Republic"] <- "Slovakia"
student2012.sub$name[student2012.sub$name == "Russian Federation"] <- "Russia"
student2012.sub$name[student2012.sub$name == "Hong Kong-China"] <- "Hong Kong S.A.R."
student2012.sub$name[student2012.sub$name == "China-Shanghai"] <- "China"

# Only need one of the plausible values, checked they are effectively
# identical
student2012.sub$PV1MATH <- as.numeric(student2012.sub$PV1MATH)
student2012.sub$PV1MACC <- as.numeric(student2012.sub$PV1MACC)
student2012.sub$PV1MACQ <- as.numeric(student2012.sub$PV1MACQ)
student2012.sub$PV1MACS <- as.numeric(student2012.sub$PV1MACS)
student2012.sub$PV1MACU <- as.numeric(student2012.sub$PV1MACU)
student2012.sub$PV1MAPE <- as.numeric(student2012.sub$PV1MAPE)
student2012.sub$PV1MAPF <- as.numeric(student2012.sub$PV1MAPF)
student2012.sub$PV1MAPI <- as.numeric(student2012.sub$PV1MAPI)
student2012.sub.math <- summarise(group_by(student2012.sub[, c(1, seq(9, 48, 
    5))], name), math = mean(PV1MATH), mCC = mean(PV1MACC), mCQ = mean(PV1MACQ), 
    mCS = mean(PV1MACS), mCU = mean(PV1MACU), mPE = mean(PV1MAPE), mPF = mean(PV1MAPF), 
    mPI = mean(PV1MAPI))
colnames(student2012.sub.math)[-c(1:2)] <- c("Change", "Quantity", "Spatial", 
    "Data", "Employ", "Formulate", "Interpret")

# Left join to only get countries that are measured
student2012.sub.map <- left_join(student2012.sub.math, world.polys)
```

```
## Joining by: "name"
```

```r
# qplot(X1, X2, order=order, group=group, data=student2012.sub.map,
# geom='polygon', fill=math) + coord_map() + new_theme_empty

# Really need all boundaries, doesn't quite work to have boundaries with one
# data: coord_map is the problem
ggplot(data = world.polys) + geom_path(aes(x = X1, y = X2, order = order, group = group), 
    colour = I("grey70")) + geom_polygon(data = student2012.sub.map, aes(x = X1, 
    y = X2, order = order, group = group, fill = math)) + # coord_map() +
new_theme_empty + theme(legend.position = "none")
```

![plot of chunk maps](figure/maps1.png) 

```r

# Now try to do an insert Note China is represented just by Shanghai
student2012.sub.math$name <- factor(student2012.sub.math$name, levels = student2012.sub.math$name[order(student2012.sub.math$math)])
p1 <- ggplot(data = world.polys) + geom_path(aes(x = X1, y = X2, order = order, 
    group = group), colour = I("grey70")) + geom_polygon(data = student2012.sub.map, 
    aes(x = X1, y = X2, order = order, group = group, fill = math)) + new_theme_empty + 
    theme(legend.position = "none")
p2 <- qplot(name, math, data = student2012.sub.math, colour = math, ylab = "Math Score", 
    xlab = "") + coord_flip() + theme(legend.position = "none")
p2 = ggplotGrob(p2)
p1 + annotation_custom(grob = p2, xmin = -40, xmax = 80, ymin = -110, ymax = 10)
```

![plot of chunk maps](figure/maps2.png) 



```r
library(YaleToolkit)
```

```
## Loading required package: lattice
## Loading required package: vcd
## Loading required package: MASS
## 
## Attaching package: 'MASS'
## 
## The following object is masked from 'package:dplyr':
## 
##     select
## 
## Loading required package: colorspace
## Loading required package: barcode
## Loading required package: gpairs
```

```r
gpairs(student2012.sub.math[, -1])
```

![plot of chunk pairs](figure/pairs.png) 

```r
# Should do a PCA, to look at what countries do better on what types of math
# skills
```
