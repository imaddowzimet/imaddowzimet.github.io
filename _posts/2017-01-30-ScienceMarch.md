---
title: "March for Science"
author: "Isaac Maddow-Zimet"
published: true
status: publish
tags: R
layout: post
---
<style type="text/css">
 
body, td {
   font-size: 14px;
}
code.r{
  font-size: 12px;
}
pre {
  font-size: 20px
}
</style>
 

 
## Open-Source Signs for the March for Science
 
The [March for Science](https://www.marchforscience.com/) doesn't have a date yet, but recently I've been channeling my energies into making science-appropriate signs for the march. And because it's a march for science, I thought it would only be appropriate that the signs be open-source and reproducible. The signs (and all the R code used to make them!) are below -- feel free to copy it, use it, or modify it, and if you have other signs to suggest, let me know and I'll add them to the post (with all appropriate credit, of course).
 
**THIS IS NOT NORMAL sign (with distribution)**
 

{% highlight r %}
# THIS IS NOT NORMAL sign (with distribution)
distribution <- rnbinom(100000, 2, .01)       # Choose distribution (generate whatever you want!)
par(family="serif", bg="#ade0e6")             # Set font and background color
par(mar=c(1, 1, 1, 1), oma=c(0,0,4,0))        # Set margins around graph (the "4" in outer margins 
                                              # is to leave room for the sign title)
 
plot(density(distribution),                   # plot the graph 
     main="", xlab="", xaxt='n', ylab="",     # with no labels
     yaxt='n', lwd=2)                         # and twice the normal line thickness
 
grid(nx = 10, ny = 10, col = "gray",          # add dotted gray grid to background 
     lty = "dotted",lwd = par("lwd"), 
     equilogs = TRUE)
 
title(main="THIS IS NOT NORMAL",              # Add title. 
      outer=TRUE, cex.main=3)                 # Make it big.
{% endhighlight %}

![plot of chunk unnamed-chunk-1](/figures/unnamed-chunk-1-1.png)
 
 
<br><br><br><br>
 
**RESIST sign (with ohm symbol)**

{% highlight r %}
par(mar = c(0,0,0,0), family="serif", bg="white")       # Set font and background color
plot(c(0, 1), c(0, 1), ann = F, bty = 'n', type = 'n',  # Plot an empty square
     xaxt = 'n', yaxt = 'n')
text(.5,.65, expression(Omega), cex=22)                 # Place the Omega sign (you may need to futz                                                           with the x and y positions)
text(.5,.2, "RESIST", cex=8)                            # Place the text
{% endhighlight %}

![plot of chunk unnamed-chunk-2](/figures/unnamed-chunk-2-1.png)
 
After I made this I found out that not only is this the symbol for the [well-known unit of resistance](https://en.wikipedia.org/wiki/Ohm), but that it was also a common symbol of the [resistance movement](https://en.wikipedia.org/wiki/Omega) [against the Vietnam draft](http://www.ebay.com/itm/272339661127). Of course it's also [Darkseid's](https://en.wikipedia.org/wiki/Darkseid) symbol, but what are you going to do. 
 
<br><br><br><br>
 
**RESIST sign (with ohm meter symbol)**  
(I thought this one was more clearly a unit of electrical resistance, as opposed to just a greek Omega, but I'm by no means an expert here, so ask an electrical engineer before you use it).
 

{% highlight r %}
# RESIST sign (with ohm meter symbol)
par(mar = c(0,0,0,0), family="serif", bg="white")
plot(c(0, 1), c(0, 1), ann = F, bty = 'n', type = 'n', xaxt = 'n', yaxt = 'n')
text(.35,.75, expression(Omega), cex=15)
text(.55,.65, expression(paste(""%.%"")), cex=15)
text(.7,.67, "m", cex=11)
text(.5,.4, "RESIST", cex=5)
{% endhighlight %}

![plot of chunk unnamed-chunk-3](/figures/unnamed-chunk-3-1.png)
 
For this last one, in particular, you might need to spend some time positioning things right, mainly because I couldn't figure out how to embed a dot product sign within one expression (you would think it would be straightforward, but I ran into some trouble getting it to look right). So this is a bit cobbled together - the omega, dot product and m are each placed separately and then aligned by eye. 
 
 
 