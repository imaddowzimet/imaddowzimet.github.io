---
title: "Mapping 2010 Census mail return rates by borough"
author: "Isaac Maddow-Zimet"
published: true
status: publish
tags: R
layout: post
img: /figures/unnamed-chunk-6-1.svg
excerpt: ""
---
 
<img src="/figures/unnamed-chunk-6-1.svg" alt="Census return rates by borough, 2010">
 
I was at the Annual Meeting of the Population Association of America a few weeks ago, and while I was there, saw a great presentation by [Joseph](http://nymag.com/daily/intelligencer/2018/03/nycs-demographer-in-chief-is-worried-about-the-2020-census.html) [Salvo](https://www.nytimes.com/2010/03/02/nyregion/02experience.html), New York City's  Demographer-in-Chief (!). He showed some pretty incredible maps of Census mail return rates (the proportion of households who return the census without follow-up), and I've been trying to recreate them in R ever since. Mail return rates are important because for every household that doesn't respond, the Census has to send fieldworkers to follow up -- and because the Census has finite resources, very low return rates increase the chance of undercounting people in specific areas (which in turn leads to less funding for resources in that area, less representation in congress, and a lot of other cascading bad effects).
 
The Center for Urban Research at CUNY has actually made a pretty fancy map of this [already](https://www.censushardtocountmaps2020.us/?latlng=37.70121%2C-91.76331&z=8&query=coordinates%3A%3A38.83543%2C-92.31262&arp=arpRaceEthnicity) by census tract, but I find maps of census tracts a *teensy* bit hard to read, so I wanted to try to visualize this at a higher level of aggregation (like neighborhood), even if I lost a bit of detail.
 
Luckily, it was very straightforward, because the Census has a pretty great administrative [dataset](https://www.census.gov/research/data/planning_database/2016/) they make available online for the purposes of survey planning, which had pretty much all of the data I needed.
 
So this is how I did it:
 
There were three datasets I ended up needing:  
  
1. The [Census 2016 planning database](https://www.census.gov/research/data/planning_database/2016/) (which has, among other things, mail response rates by census tract),  
2. A shapefile of [neighborhoods](https://www1.nyc.gov/site/planning/data-maps/open-data/dwn-nynta.page) from the NYC Department of Planning,   
3. And a [crosswalk](https://www1.nyc.gov/site/planning/data-maps/open-data/dwn-nynta.page) between the two of them.
 
Here's the code for loading the data (along with a bit of cleaning):  
{% highlight r %}
library(tidyverse)
library(rgdal)
library(maptools)
library(ggthemes)
library(proj4)
library(viridis)
gpclibPermit() 
 
# Load shapefile (change the filepath to where you have it stored)
neighborhoods.shape <- readOGR(dsn = "/Users/isaacmaddowzimet/Desktop/Census/nynta_18a", 
                               layer = "nynta")
 
# Transform to a more normal projection
neighborhoods.shape2 <- spTransform(neighborhoods.shape, CRS("+proj=longlat +datum=WGS84"))
 
# Convert shapefile to dataset
neighborhoods <- fortify(neighborhoods.shape2, region = "NTAName")
 
# Load crosswalk
crosswalk <- read.csv("/Users/isaacmaddowzimet/Desktop/Census/nta_census_crosswalk.csv")
 
## Load planning database, and limit to just NYC: ##
 
# List NYC counties (so we can limit the dataset to just those 5 counties) 
NYCcounties <- c(5, 47, 61, 81, 85)
 
# Select only those five counties, and select only the variables we need
censusdataNY <- read.csv("/Users/isaacmaddowzimet/Desktop/Census/pdb2016trv8_us.csv") %>%
  filter((County %in% NYCcounties) 
         & (State_name == "New York"))  %>%
  modify_if(is.factor, factor)          %>% 
  select(State, State_name, County, 
         County_name, Tract,
         Mail_Return_Rate_CEN_2010, 
         GIDTR, Valid_Mailback_Count_CEN_2010, 
         FRST_FRMS_CEN_2010, RPLCMNT_FRMS_CEN_2010) 
{% endhighlight %}
   
The census dataset has an already calculated mail return rate variable, but since I was going to need to recalculate it at the neighborhood level, I wanted to make sure I could recreate it from the raw variables in the dataset.  The denominator (number of valid, occupied addresses in the census tract) was pretty clearly identified in the codebook, but the numerator was a bit less clear, so I ended up guessing and adding up **FRST_FRMS_CEN_2010** (Number of housing units that returned initial form in 2010 Census) and **RPLCMNT_FRMS_CEN_2010** (Number of housing units that returned replacement form in 2010 Census). I then checked to make sure they matched the calculated rate:  
 

{% highlight r %}
censusdataNY <- mutate(censusdataNY, Return_Rate_Calc = ((FRST_FRMS_CEN_2010+RPLCMNT_FRMS_CEN_2010)/Valid_Mailback_Count_CEN_2010)*100)
head(select(censusdataNY, Return_Rate_Calc, Mail_Return_Rate_CEN_2010))
{% endhighlight %}



{% highlight text %}
##   Return_Rate_Calc Mail_Return_Rate_CEN_2010
## 1               NA                        NA
## 2         68.14159                      68.1
## 3         70.92008                      70.9
## 4         74.92013                      74.9
## 5         63.42857                      63.4
## 6         74.25109                      74.3
{% endhighlight %}
   
And magically, it did! (this almost never happens). (also, don't worry, I did a much more systematic check of this, just not showing it here because it is extremely boring).  
  
Next step was to merge the dataset with the crosswalk file:    

{% highlight r %}
# Create parallel census tract identifier by combining NYS code, county code and census tract
crosswalk <- crosswalk %>% 
  mutate(GIDTR = 36000000000 + 
        (X2010.Census.Bureau.FIPS.County.Code*1000000) +
         X2010.Census.Tract) %>%
  rename(id = NTA)
 
# Some other clean-up to make the merge slightly more painless
crosswalk$id <- as.character(crosswalk$id) 
crosswalk$id[crosswalk$id == "Hudson Yards-Chelsea-Flat Iron-Union Square"] <- "Hudson Yards-Chelsea-Flatiron-Union Square"
 
# Merge away!
censusdataNYwNTA <- inner_join(censusdataNY, crosswalk, by= "GIDTR")
{% endhighlight %}
  
And then calculate return rates by neighorhood (instead of census tract) with the magic of the **dplyr** package.
 

{% highlight r %}
# Change all NAs to 0s (in numerator and denominator) for missing census tracts
# (this is so we can avoid setting the whole neighbrhood to missing when 
# just one census tract is missing; later on we'll set neighborhoods composed of
# entirely missing census tracts back to NA)
censusdataNYwNTA$FRST_FRMS_CEN_2010[is.na(censusdataNY$Return_Rate_Calc)] <- 0
censusdataNYwNTA$RPLCMNT_FRMS_CEN_2010[is.na(censusdataNY$Return_Rate_Calc)] <- 0
censusdataNYwNTA$Valid_Mailback_Count_CEN_2010[is.na(censusdataNY$Return_Rate_Calc)] <- 0
 
# Define function to calculate rate
calc.rate <- function(num1, num2, denom) {
  
  rate <- ((sum(num1)+sum(num2))/sum(denom))*100
  
}
 
# Calculate return rate by neighborhood, collapse into neighborhood dataset
censusdataNYneighborhoods <- censusdataNYwNTA %>%
  group_by(id, Borough) %>%
  summarise( mail_return_rate_calc = calc.rate(FRST_FRMS_CEN_2010, RPLCMNT_FRMS_CEN_2010, Valid_Mailback_Count_CEN_2010))
 
# Neighborhoods with return rates of 0 should be set back to missing
censusdataNYneighborhoods$mail_return_rate_calc[censusdataNYneighborhoods$mail_return_rate_calc==0] <- NA
{% endhighlight %}
 
Our dataset now looks like this, with one row per neighborhood:
 

{% highlight r %}
head(censusdataNYneighborhoods)
{% endhighlight %}



{% highlight text %}
## # A tibble: 6 x 3
## # Groups:   id [6]
##   id                                         Borough    mail_return_rate_…
##   <chr>                                      <fct>                   <dbl>
## 1 Airport                                    Queens                  NaN  
## 2 Allerton-Pelham Gardens                    Bronx                    66.1
## 3 Annadale-Huguenot-Prince's Bay-Eltingville Staten Is…               76.4
## 4 Arden Heights                              Staten Is…               75.9
## 5 Astoria                                    Queens                   69.3
## 6 Auburndale                                 Queens                   77.2
{% endhighlight %}
  
Now we can plot! It's very easy to do this for all of New York City:  
 

{% highlight r %}
plotdata <- inner_join(neighborhoods, censusdataNYneighborhoods, by= "id")
 
sc <- scale_fill_gradientn(colours = viridis(20), limits = c(54,85), breaks = c(60, 70, 80))
 
ggplot() + 
  geom_polygon(data=plotdata, 
               aes(x= long, y=lat, group=group, fill = mail_return_rate_calc)) + 
  theme_tufte() + 
  coord_map() + ggtitle("Mail return rates by neighborhood, Census 2010") +
  sc +
  guides(fill = guide_colorbar(ticks=F, title="Percent of \nforms returned", barwidth = 1, barheight = 6)) + 
  theme(axis.line=element_blank(),axis.text.x=element_blank(),
          axis.text.y=element_blank(),axis.ticks=element_blank(),
          axis.title.x=element_blank(),
          axis.title.y=element_blank(), 
        plot.title = element_text(hjust=.5, face="bold"))
{% endhighlight %}

<img src="/figures/unnamed-chunk-6-1.svg" title="plot of chunk unnamed-chunk-6" alt="plot of chunk unnamed-chunk-6" style="display: block; margin: auto;" />
  
I'm using the [viridis](https://cran.r-project.org/web/packages/viridis/vignettes/intro-to-viridis.html) package here, which is colorblind friendly, and all around awesome.   

It's also easy to split these maps up by borough (note that the scales are different for each, so we can see relative differences within boroughs). I'm not reproducing the code here, because it's a bit clunky, but all I'm doing is mapping with different subsets of the data:  
 
<img src="/figures/unnamed-chunk-7-1.svg" title="plot of chunk unnamed-chunk-7" alt="plot of chunk unnamed-chunk-7" style="display: block; margin: auto;" /><img src="/figures/unnamed-chunk-7-2.svg" title="plot of chunk unnamed-chunk-7" alt="plot of chunk unnamed-chunk-7" style="display: block; margin: auto;" /><img src="/figures/unnamed-chunk-7-3.svg" title="plot of chunk unnamed-chunk-7" alt="plot of chunk unnamed-chunk-7" style="display: block; margin: auto;" /><img src="/figures/unnamed-chunk-7-4.svg" title="plot of chunk unnamed-chunk-7" alt="plot of chunk unnamed-chunk-7" style="display: block; margin: auto;" /><img src="/figures/unnamed-chunk-7-5.svg" title="plot of chunk unnamed-chunk-7" alt="plot of chunk unnamed-chunk-7" style="display: block; margin: auto;" />
  
I don't neccesarily have anything to say about these maps -- just think they are interesting, especially in the context of planning for the 2020 Census, which by all accounts is going to be much more difficult to collect data for (because of the proposed -- and incredibly ill-advised -- citizenship question, for one). People already don't respond to the Census for all sorts of reasons -- distrust of the government, confusion about who needs to fill out the form (response rates are much lower in neighborhoods with higher proportions of renters, for example), busyness -- and it's not a great idea to add another reason for people not to respond. You can also see that response rates are are already incredibly variable by neighborhood - and while a lot of follow-up happens, it's not hard to think that areas that require more follow-up have a higher risk of being undercounted (and thus underrepresented and underfunded).  
 
 
 
 
