---
title: "Changes in contraceptive method mix over time"
author: "Isaac Maddow-Zimet"
published: true
status: publish
tags: R
layout: post
img: https://github.com/imaddowzimet/imaddowzimet.github.io/raw/master/methodmix.png
excerpt: ""
---
 

 
![](https://github.com/imaddowzimet/imaddowzimet.github.io/raw/master/methodmix.png)
 
**Preamble**  
  
I was having fun playing with stacked area graphs in my last [post](https://imaddowzimet.github.io/MoMApart2/), so I thought for this one I would extend them to a topic area that's much more in my wheelhouse: contraception. There's a large survey that I work with a lot - the National Survey of Family Growth (NSFG) - that tracks what contraceptive methods people use, and since it's administered at fairly regular periods, you can use it to track shifts over time.  It's a nice one to use because if you make the denominator contraceptive users, the percent of people using each method adds up to 100.  I also think the chart (as you'll see) is a nice case study in A) how to deal with data that has a ridiculous number of categories, and B) how to graphically convey situations where you only have data at specific points in time, but want to show an estimated trend.   
 
**The Good Stuff (the code)**
 
So I first load some packages - we'll get into them later -- but they're essentially to help me read the data, do a bit of imputation (in this case, to estimate the trend where I don't have data), and plot.

{% highlight r %}
########################
# SET-UP
#######################
library(readxl)
library(plotrix)
library(imputeTS)
{% endhighlight %}
 
I'm getting the data from published NSFG reports, as opposed to running anything from the dataset directly -- mainly because it's a lot more straightforward for the purposes of this post. The data are from Table 4 of [this report](https://www.cdc.gov/nchs/data/series/sr_23/sr23_029.pdf) for the years 1982, 1995, and 2002, Table 1 of [this report](https://www.cdc.gov/nchs/data/ad/ad182.pdf) for 1988, and and Table 1 of [this report](https://www.cdc.gov/nchs/data/nhsr/nhsr086.pdf) for 2008 and 2012. I used [Tabula](http://tabula.technology/), one of the most amazing pieces of software ever created, to extract the data tables in the PDF reports. These tables all describe current use of a method (in other words, the method the respondent said they were using during the month they were interviewed), among women aged 15-44.  It doesn't take into account whether the respondent was sexually active that month, or why they were using contraception - it's really just a description of what methods women were using at a snapshot in time.  
 

{% highlight r %}
# Load data (you'll need to change the filepath if you download the data)
data <- as.data.frame(read_xls("/Users/isaacmaddowzimet/imaddowzimet.github.io/method mix 1982-2012.xls", sheet="Recode 1"))
 
# Change NAs to 0s (in some years, some methods didn't exist yet)
data[is.na(data)] <- 0
{% endhighlight %}
 
The data looks like this, with method names as rows, and years as columns (I'm only showing a few years here so the table will fit):

{% highlight r %}
print(data[,1:4])
{% endhighlight %}



{% highlight text %}
##                                         Method 1982 1988 1995
## 1                         Female sterilization 12.9 16.6 17.8
## 2                           Male sterilization  6.1  7.0  7.0
## 3                                         Pill 15.6 18.5 17.3
## 4                   Implant, Lunelle, or patch  0.0  0.0  0.9
## 5                           3-month injectable  0.0  0.0  1.9
## 6                           Contraceptive ring  0.0  0.0  0.0
## 7                          Intrauterine device  4.0  1.2  0.5
## 8                                    Diaphragm  4.5  3.5  1.2
## 9                                       Condom  6.7  8.8 13.1
## 10         Periodic abstinence calendar rhythm  1.8  1.0  1.3
## 11 Periodic abstinence natural family planning  0.3  0.4  0.2
## 12                                  Withdrawal  1.1  1.3  2.0
## 13                               Other methods  2.7  1.3  1.1
{% endhighlight %}
 
Because I want the denominator to be contraceptive users, as opposed to all women, I prorate the percent distribution of methods to add up to 100%. Normally, I might be pretty cautious about doing something like this, except that what I'm really interested in is how women's contraceptive method mix has changed over time - not whether women are using contraception more (also, overall levels of contraceptive use actually didn't change that drastically between 1988 and 2012). I then transpose the dataset (so that years are rows and methods are columns), because that's how *stackpoly*, the command we're going to use to make the stacked area chart, wants to see the data. 

{% highlight r %}
# Rescale to sum to 100
data.stand <- data
standardize <- function(x) {x*(100/sum(x))}
data.stand[,-1] <- sapply(data.stand[,-1], standardize)
 
# Tranpose the data
n <- data.stand$Method                              # This just stores the method names, since we'll use them as column names later on
datatranspose <- as.data.frame(t(data.stand[,-1]))  # This transposes, excluding the method column
colnames(datatranspose) <- n                        # And this brings back in the column names (the methods)
 
# Print first three columns of the data
print(datatranspose[,1:3])
{% endhighlight %}



{% highlight text %}
##      Female sterilization Male sterilization     Pill
## 1982             23.15978          10.951526 28.00718
## 1988             27.85235          11.744966 31.04027
## 1995             27.68274          10.886470 26.90513
## 2002             26.93548           9.193548 30.48387
## 2008             26.52733           9.967846 27.49196
## 2012             25.12156           8.265802 25.93193
{% endhighlight %}
 
The problem is that now we still only have data points for the years with data. This would be fine if we wanted to do a set of stacked bar charts, but sometimes it's nice to get a sense of overall trends (and stacked bar charts can be misleading too, because they force the data years to be evenly spaced - which can make some trends look much sharper than they actually are).So we need to interpolate, which is basically using the information from the data we have to guess at the data we don't have.  Often people use linear interpolation, which is essentially just drawing a straight line between points; I'm using spline interpolation here, which uses a bit of fancy (but not that fancy) math to draw curved lines between points, guessing at the correct shape of the overall curve based on the information we have (that sound is 10 million mathematicians rolling over in their graves at my garbled definition).
 
Codewise, though, it's pretty simple. I basically add extra rows for the extra years, tell R that it is a time series, and then use a cool interpolation function that comes with the *imputeTS* package, which I loaded above. Thank god that this was written already, since I spent a day failing to code it from scratch (which I think has to do with me being dumb at coding, not it being so complicated) before I realized that someone already had.  
 

{% highlight r %}
# Add extra years
x <- setdiff(1982:2012, c(1982,1988,1995,2002,2008,2012)) # the second argument excludes years we already have
extrarows <- matrix(NA, length(x), dim(datatranspose)[2]) # puts the list in a matrix
rownames(extrarows) <- x                         
colnames(extrarows) <- n
datafull <- rbind(datatranspose, extrarows)             # Appends it to our existing data
datafull <- datafull[order(row.names(datafull)),]       # Orders it by year
 
# Convert to time series
datafull.ts <- ts(datafull,start=1982)                  # It's that easy!
# Interpolate
datafull.ts.interp <- na.interpolation(datafull.ts, option="spline") # It's also that easy!
{% endhighlight %}
 
Finally, we make the graph! I spent a lot of time futzing with the colors, because there are 13 categories of contraceptive methods (which is way too many), but I didn't want to collapse any, because part of what's interesting to me is how methods that were once more common (like the diaphragm) faded out over time. Instead, I tried to find similar colors for broad categories (purple for sterilization, green for long acting methods, for example), and then mess with the shading for the specific methods for each broad category. I also tried to very hard to pick colors that didn't look like a 1992 Windows desktop screensaver.
 
The other thing I did was to add dotted lines for the years with actual data (well, except for the beginning and end years), to try to make clear what the precise shape of the curves is a bit of a guess. With line graphs, I've often seen people add points to the line that denote `actual' values, but with a stacked area chart that starts to look crazy fast. I think this is pretty successful (and I like the way it looks, but would be curious to hear other solutions people have used.) Also to try to make it extra clear, I only added x-axis labels for years with actual data.    
  
So without further ado, the graph! I'm not going to do any interpretation here, because I think part of the fun of this kind of dense graph is exploring it on your own -- as always, leave any feedback (or errors in my code!) in the comments.
 

{% highlight r %}
# Convert it back to a data frame (because stackpoly doesn't like time series)
finaldata <- as.data.frame(datafull.ts.interp)
 
# Change a few column names for legend purposes
colnames(finaldata)[c(4,10,11)] <- c("Implant/Lunelle/patch","Rhythm", "Natural family planning")
 
# Define colors 
legendcolors <- c("#4f114c","#821c7e","#1b5c82", "#21821b", 
                  "#1a6b15", "#145110", "#0d380b", "#eded2a", 
                  "#efcc2d", "#772514", "#ad361f", 
                  "#e21f1f","#a09292")
 
# Put some extra space on the right side for the big legend
par(oma=c(0,0,1,11))
 
# Only add x-axis labels for years we have data for
xlab <- c(1982,NA,NA,NA,NA,NA,1988,NA,NA,NA,NA,NA,NA, 
          1995,NA,NA,NA,NA,NA,NA,2002,NA,NA,NA,NA,NA,2008,
          NA,NA,NA,2012)
 
# Plot!
stackpoly(finaldata, stack=TRUE, ylim=c(0,100), xaxlab = xlab, col=legendcolors, axis4 = FALSE)
 
# Add legend!
legend(32, 100, legend=rev(colnames(finaldata)), fill=rev(legendcolors), 
       xpd=NA, horiz=FALSE, cex=1.1, bty='n',
       title=expression(bold("Method")) , title.adj=0.09, y.intersp=1.3)
 
# Add title!
mtext(expression(bold(" Contraceptive method mix among contraceptive users \n aged 15-44, 1982-2012")), outer=TRUE, line=-2.1, adj=.55, cex=1.2)
 
# Add dotted lines for actual data
abline(v=c(7,14,21,27), lty=3, lwd=2)
 
# Add footnote
mtext("Note: trends estimated using cubic spline; dotted lines denote years with data", outer=TRUE, cex=.9, side=1, line=-2)
{% endhighlight %}

<img src="/figures/unnamed-chunk-6-1.svg" title="plot of chunk unnamed-chunk-6" alt="plot of chunk unnamed-chunk-6" style="display: block; margin: auto;" />
  
 
 
 
 
