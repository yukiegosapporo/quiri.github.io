---
layout: post
title: "xkcd survey and the power to shape the internet"
excerpt: "Randall breaks the internet (a little)" 
categories: articles
tags: [R, xkcd, Google Trends]
comments: true
share: true
published: true
author: safferli
modified: 2015-09-03
---


## The xkcd survey

<span class = "dropcap">I</span>f you've never heard of xkcd, it's ["[a] webcomic of romance, sarcasm, math, and language"](http://www.xkcd.com/) created by [Randall Munroe](https://en.wikipedia.org/wiki/Randall_Munroe). Also, if you've never heard of xkcd, be prepared for losing at least a day's worth of productivity reading the comics and the excellent [what if](http://what-if.xkcd.com/) column where Randall answers hypothetical questions with physics. 

Randall used his webcomic to post a survey, asking the internet to fill out a number of questions designed to generate "an interesting and unusual data set for people to play with":

![xkcd survey webcomic call to action](http://imgs.xkcd.com/comics/xkcd_survey.png)
{: .center}

If you're reading this, please head over to [the survey](http://xkcd.com/1572/) and fill it out! Also, please send it to your friends and acquaintances who usually do not read these kind of blogs and webcomics -- the more diverse the audience, the more interesting the resulting data set! 

While the survey is still open, the resulting dataset is not available. But can we already do some nifty and fancy still with it? Yes we can! 


## The internet

The internet is a pretty [big place](http://internet-map.net/). So big in fact, it's pretty difficult for an individual to affect it. Granted, Michael Jackson [crashed google](http://edition.cnn.com/2009/TECH/06/26/michael.jackson.internet/) upon his death. But most of us do not have his reach, and we're all pretty much alive still. 

A single person *can* have an impact on the internet as whole. Either you're as famous as Michael, or you can bring your weight to bear on a particularly obscure and unknown part of the internet. In this case, your *relative* impact (and economists always talk in relative terms) is much higher. 

Randall has a pretty large impact on the internet to start with (well, compared to me; not compared to Michael Jackson), but is it large enough to move even Google Search Trends, or in other words, to move the collective internet conciousness? 


## The first-look analysis

This is just a first look at the data, but I wanted to get this out as soon as possible, to convince more people to take part in the survey. Hopefully by seeing that you can indeed do interesting stuff with the survey -- before the actual data has been released even! -- more people will be convinced to give it a try. 

Google Search Trends have been used by many, for instance for [unemployment forecasting](http://ejournals.duncker-humblot.de/doi/abs/10.3790/aeq.55.2.107), to detect [influenza epidemics](http://www.nature.com/nature/journal/v457/n7232/full/nature07634.html), and to monitor [suicide rates](http://www.sciencedirect.com/science/article/pii/S0165032709003978). The idea is always the same: if something has a large enough impact, it will shift the google search trends upwards. 

In the xkcd survey, one question was if the reader knew the meaning of a number of obscure English words, such as "Regolith", or "Slickle". My hypothesis is that the interest in these words by people reading the survey will be piqued, and thus these readers will google for these words -- leading to a tiny, tiny uplift in the search trends for the respective word. If Randall's reach on the internet is large enough, we should be able to see a noticeable uplift in the search trends for these words. 

The second hypothesis is that the relative impact on more obscure words is larger, so we should see a large uplift in more esoteric words. This is untested here, since I do not have a good idea of how to rank words by "obscureness". If someone has a smart idea on this, please contact me, I'd be delighted!

I grabbed the search trends data on the xkcd survey words off the [google search trends site](http://www.google.com/trends/explore) as .csv files. The data is available on github along with the full code needed to process it. Reading the data is relatively straightforward, you just have to account for the fact that google only allows you to search for five keywords on the website (resulting in four csv files), and that the csv is filled with a lot information that you don't need, so I just pull the relevant rows and merge them all into one dataset: 

{% highlight r %}
f.gtrend.csv.read <- function(x){
  read.csv(file=paste0("data/report_", sprintf("%02d", as.numeric(x)), ".csv"), 
           na.strings = " ", 
           stringsAsFactors = FALSE,
           # 173 is last timeseries row, minus skip, minus header = 168
           skip = 4, nrows = 168)
}

# merge all four google .csvs 
data.raw <- f.gtrend.csv.read(1) %>% 
  merge(f.gtrend.csv.read(2)) %>% 
  merge(f.gtrend.csv.read(3)) %>% 
  merge(f.gtrend.csv.read(4)) %>% 
  mutate(
    Time = as.POSIXct(strptime(Time, "%Y-%m-%d-%H:%M UTC", tz="UTC"))
  )
{% endhighlight %}

Now, with the data available in R, let's graph this stuff! 

{% highlight r %}
# big pile of trend lines, starting September-01
data.raw %>% 
  gather(term, trend, -Time) %>% 
  ggplot()+
  geom_line(size=1)+
  aes(x=Time, y=trend, colour=term)+
  coord_cartesian(xlim = c(as.POSIXct(strptime("2015-09-01", "%Y-%m-%d", tz="UTC")),
                           max(data.raw$Time)))+
  ggtitle("Google Search Trends of the xkcd survey English Terms")
{% endhighlight %}

![xkcd trending words, messy]({{ site.url }}/images/safferli/xkcd-messy-trends.png)

Oof. A mess of lines. Let's get that sorted out by plotting each search term individually: 

{% highlight r %}
# faceted for better overview
data.raw %>% 
  gather(term, trend, -Time) %>% 
  ggplot()+
  geom_line(size=1)+
  aes(x=Time, y=trend, colour=term)+
  coord_cartesian(xlim = c(as.POSIXct(strptime("2015-09-01", "%Y-%m-%d", tz="UTC")),
                           max(data.raw$Time)))+
  facet_wrap(~term)+
  guides(colour=FALSE)+
  theme(axis.text.x=element_text(angle = 45, hjust = 1))+
  ggtitle("Google Search Trends of the xkcd survey English Terms")
{% endhighlight %}

![xkcd trending words, faceted]({{ site.url }}/images/safferli/xkcd-facet-trends.png)

Much nicer. We can clearly see that some words (e.g. unitory, regolith, fination) show a clear uplift after the survey was posted (around noon UTC, September 2nd). Other words have a smaller impact (e.g. apricity, revergent, cadine), while others again have a minuscle to zero impact (e.g. rife, soliquy having a tiny impact, and hubris having zero). 

## Conclusion

Randall *does* have an impact on the search trends! 

Taking a deeper look into which words have a higher impact will be interesting. Also, once the survey results have been posted, an obvious test would be check for interaction between the search impact and the share of people knowing the word in question. If more survey users know the word, there is no need for them to google it, and thus resulting in a smaller search trend impact. 

Again, please share the survey and pester your friends to fill it in as well -- for science! 

Code and data for this analysis is availabe on [github](https://github.com/safferli/xkcd_survey), of course. 

