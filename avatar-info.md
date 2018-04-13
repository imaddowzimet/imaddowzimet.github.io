---
layout: page
title: How did I make my avatar?
permalink: /avatar-info/
---

![](https://github.com/imaddowzimet/imaddowzimet.github.io/raw/master/images/headerimage.png)


Good question! The answer is that I deserve almost zero credit for it. I took the code from Antonio Sánchez Chinchón, who wrote a much better [blog post](https://fronkonstin.com/2018/04/04/the-travelling-salesman-portrait/) about it than I ever could, and then I totally mangled his creation by adding some colored dots, because I liked the purple shirt/black tie combo I was wearing in the picture, and wanted that to come through.  The complete code is here (you'll need to put in your own image file, but the rest should work out of the box):


{% highlight r %}
library(imager)
library(dplyr)
library(ggplot2)
library(scales)
library(TSP)
library(ggthemes)

# Load, convert to grayscale, filter image (to convert it to bw) and sample
load.image("yourimage.jpg") %>% 
  grayscale() %>%
  threshold("55%") %>% 
  as.cimg() %>% 
  as.data.frame()  %>% 
  sample_n(8000, weight=(1-value)) %>% 
  select(x,y) -> data

# load color version
colorimage <- load.image("guttmacher-214.jpg") %>% 
  as.data.frame(wide="c") %>% mutate(rgb.val=rgb(c.1,c.2,c.3), id=row_number())
  
# Compute distances and solve TSP (it may take a minute)
as.TSP(dist(data)) %>% 
  solve_TSP(method = "arbitrary_insertion") %>% 
  as.integer() -> solution

# Create a dataframe with the output of TSP
data.frame(id=solution) %>% 
  mutate(order=row_number()) -> order

# Rearrange the original points according the TSP output
data %>% 
  mutate(id=row_number()) %>% 
  inner_join(order, by="id") %>% arrange(order) %>% 
  select(x,y) -> data_to_plot

# Join with color
datacolor <- inner_join(data_to_plot, colorimage, by=c("x", "y"))

# A little bit of ggplot to plot results
ggplot(data_to_plot, aes(x,y)) +
  geom_path() + geom_point(size=.1, col=datacolor$rgb.val)+
  scale_y_continuous(trans=reverse_trans())+
  coord_fixed()+
  theme_tufte()

# Save out image
ggsave("yourimagename.png", dpi=800, width = 4, height = 4)
{% endhighlight %}

