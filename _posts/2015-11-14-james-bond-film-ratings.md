---
layout: post
title: James Bond movies
excerpt: Box office success of the Bond films
categories: articles
tags: [R, ggplot, rvest, James Bond]
comments: true
share: true
published: true
author: safferli
date: 2015-11-14T13:00:00+02:00
---

> James Bond: Do you expect me to talk?  
> Auric Goldfinger: No, Mr. Bond, I expect you to die! 

## James Bond

I'm a big James Bond fan, so naturally I went to watch the new Bond movie *Spectre* which -- spoiler alert! -- is pretty bad. It also got me to reminice about the good Bond films of the past. My personal candidate for worst Bond film is *Die Another Day*, but what does the "objective" opinion say on this hotly debated topic? Does my taste conform to the Internet's taste? 

The Economist newspaper did some data analysis when the last Bond (*Skyfall*) came out, stacking up the different Bond actors on killing, drinking martinis, and love conquests (["Booze, bonks and bodies"](http://www.economist.com/news/books-and-arts/21564816-various-bonds-are-more-different-you-think)). They [updated the data](http://www.economist.com/blogs/graphicdetail/2015/10/daily-chart-13) for the UK release of *Spectre*, with Daniel Craig jumping in ranking a lot, mainly from his many kills. The Economist also recently did a [comparison of box office opening weekends](http://www.economist.com/blogs/graphicdetail/2015/11/daily-chart-5) of the Bond films. 

As an economist, I'd have to argue that box office success is one objective measure of film quality; if the movie is bad, people don't go to the cinemas to watch it, and after all, the market is always right, right? 

Others might argue that a critical review scale can assess the quality of a film. This is always debateable: who decides what and how things are grouped into a quality scale? Nevertheless, metacritic and other ratings are used a lot in different industries, because it's the best you can get. 


## Data

I decided to pull some data on the Bond movies. Luckily, wikipedia has a nice [article](https://en.wikipedia.org/wiki/List_of_James_Bond_films), listing all bond films with their respective budget, box office returns, and several critic scales. I'll use the excellent and easy to use `rvest` package to pull the data. 

{% highlight r %}
## pull oldid of the wikipedia page to ensure reproduceability
bond.url <-  "https://en.wikipedia.org/w/index.php?title=List_of_James_Bond_films&oldid=688916363"

## read the page into R
bond.wiki <- read_html(bond.url)

## film data
bond.films <- bond.wiki %>%
  html_nodes("table") %>%
  # first table in the page
  .[[1]] %>%
  # fill, because wiki uses multi-cell formatting :(
  html_table(fill = TRUE) 
{% endhighlight %}

`rvest` really is that easy to use. It was the first time I used it, and I have to say, I like it a lot. It'll make pulling data from internet webpages much, much easier in the future for me! 

The only problem is that the table in the wiki article has a lot of extra information (like footnotes) that we now need to clean to get a nice, usable dataframe. It's relatively straightforward, I've written a couple of custom functions doing mainly some regexp cleaning. These are the `f.` functions in the code below. If you're interested in the details you can check out the full code, including the function code on [github](https://github.com/safferli/james_bond_films). 

{% highlight r %}
# lots of cleaning to work to be done now...
bond.films %<>% 
  # make clean column names (replace space with dot)
  setNames(make.names(names(bond.films), unique = TRUE)) %>% 
  # remove first and last line (inner-table headers)
  head(-1) %>% tail(-1) %>% 
  mutate(
    # clean wiki-related data import problems
    Title = f.clean.titles(Title),
    Bond.actor = f.clean.double.names(Bond.actor), 
    Director = f.clean.double.names(Director),
    # NA coerced here are what we want -- ignore the warnings
    Box.office = as.numeric(f.clean.footnote.marks(Box.office)),
    Budget = as.numeric(f.retrieve.initial.numbers(Budget)), 
    Salary.of.Bond.actor = as.numeric(f.retrieve.initial.numbers(Salary.of.Bond.actor)),
    # rename the 2005 adjusted price values
    Box.office.2005.adj = as.numeric(Box.office.1), 
    Budget.2005.adj = as.numeric(Budget.1), 
    Salary.of.Bond.actor.2005.adj = as.numeric(f.retrieve.initial.numbers(Salary.of.Bond.actor.1)), 
    RoI = Box.office.2005.adj/Budget.2005.adj
  ) %>% 
  # remove old uninformative uniqueness naming
  select(-contains("1"))
{% endhighlight %}

The `make.names`command was new for me, and I am very impressed. It makes setting up proper R-usable variable names a breeze, and in this case was also helpful with uniqueness: The wiki article page uses multi-cell formatting to identify columns, which got lost in my transformations. It's a bother people don't conform to [clean data standards](https://cran.r-project.org/web/packages/tidyr/vignettes/tidy-data.html) everywhere! &#128521;

I then pull and clean the second table as well, which has information on several awards and critic ratings, and merge both for the final dataset to use: 

{% highlight r %}
## ratings on films
bond.ratings <- bond.wiki %>%
  html_nodes("table") %>%
  # second table in the page
  .[[2]] %>%
  # fill, because of multi-cell formatting :(
  html_table(fill = TRUE)

bond.ratings %<>% 
  setNames(make.names(names(bond.ratings), unique = TRUE)) %>% 
  mutate(
    # rename for later merging
    Title = f.clean.titles(Film),
    Actor = f.clean.double.names(Actor) 
  ) %>% 
  # reorder to original order
  select(Title, Year:Awards) %>% 
  # split the RT into rating and number of reviewers
  separate(Rotten.Tomatoes, c("Rotten.Tomatoes.rating", "Rotten.Tomatoes.reviews"), sep="%") %>% 
  mutate(
    Rotten.Tomatoes.reviews = sub("^ \\((\\d*) reviews\\).*", "\\1", Rotten.Tomatoes.reviews)
  )

## final data set
bond.dta <- bond.films %>% 
  merge(bond.ratings) %>% 
  select(-Actor) %>% 
  arrange(Year)
{% endhighlight %}

The [Rotten Tomatoes](https://en.wikipedia.org/wiki/Rotten_Tomatoes) rating is the one I will be using, by virtue of the fact that it's available for all films. 


## Graphing

With all the data pulled, let's take a look at it more closely. 

{% highlight r %}
bond.dta %>% ggplot()+
  geom_point(aes(x=Year, 
                 y=Box.office.2005.adj, 
                 size=Budget.2005.adj, 
                 colour=as.numeric(Rotten.Tomatoes.rating)
            ))+
  scale_colour_continuous()
{% endhighlight %}

![Bond movies, box office sales, barebones]({{ site.url }}/images/safferli/bond-bare-plot.png)

A little messy, but you can already get the general idea. There was a sharp decline in box office earnings in the late 1980s, and the older Bond movies (with Connery as Bond in particular) have better ratings. 

We'll clean this graph a bit more, and also add the Bond actors. For that, we need to generate a dataset of Bond actors and their time of service. 

{% highlight r %}
actor.grp <- bond.dta %>% 
  group_by(Bond.actor) %>% 
  summarise(
    yearmin = min(Year), 
    yearmax = max(Year)
  ) %>% 
  ungroup() %>% 
  arrange(yearmin) %>% 
  # George Lazenby only had one film, which makes geom_rect "invisible"
  mutate(
    yearmax = ifelse(yearmin==yearmax, yearmax+1, yearmax)
  )
{% endhighlight %}

With this information, we can build a large `ggplot` graph with all the information in one place!

{% highlight r %}
ggplot() + 
  # place geom_rect first, then other geoms will "write over" the rectangles
  # write a background rectangle for each Bond actor years of service
  geom_rect(data = actor.grp, aes(xmin = yearmin, xmax = yearmax, 
                                  ymin = -Inf, ymax = Inf, 
                                  fill = Bond.actor), alpha = 0.3)+
  # write actor names on rectangles
  geom_text(data = actor.grp, aes(x = yearmin, 
                                  # place text rather at the top of the y-axis
                                  y = max(bond.dta$Box.office.2005.adj, na.rm = TRUE), 
                                  label = Bond.actor,
                                  # colour is already mapped to RT.rating (continuous)
                                  # fill=Bond.actor,
                                  angle = 90, 
                                  hjust = 1, vjust = 1
                                  ), alpha = 0.6, size = 5)+
  # film names
  geom_text(data = bond.dta, aes(x = Year, y = 0, 
                                 label = Title, 
                                 angle = 90, 
                                 hjust = 0, vjust = 0.5), size=4)+
  # film data
  geom_point(data = bond.dta, aes(x = Year, 
                                  y = Box.office.2005.adj, 
                                  size = Budget.2005.adj, 
                                  colour = as.numeric(Rotten.Tomatoes.rating)))+
  # Rotten Tomatoes rating gradient
  scale_colour_continuous(low="red", high="green", name = "Rotten Tomatoes rating")+
  # increase minimum point size for readability
  scale_size_continuous(name = "Budget (2005 mil. dollars)", range = c(3, 10))+
  theme_bw()+
  theme(plot.title = element_text(lineheight=.8, face="bold"))+
  # remove actor names from legend
  guides(fill=FALSE)+
  labs(title = "Box office results, budgets, and ratings of James Bond films\n", 
       x="", y="Box office earnings (in 2005 mil. dollars)")
# export to size that fits everything into graph, use golden ratio
ggsave(file="bond-full.png", width = 30, height = 30/((1+sqrt(5))/2), units = "cm")
{% endhighlight %}

![xkcd trending words, messy]({{ site.url }}/images/safferli/bond-full.png)



## Results

It turns out that my personal least favourite Bond isn't that bad by the scales provided above: it was relatively successful at the Box office, and also wasn't rated *that* badly. 

The honour of the worst-rated Bond film goes to *A View to a Kill* (Roger Moore), and the film with the worst box office results is *License to Kill* (Timothy Dalton). Perhaps it's the naming policy? It seems you do not make a killing with Bond movies if they have the word "kill" in the title -- these two are the only ones. 

Roger Moore presided over a continuous decline in popularity in the 1980s, and Timothy Dalton could not stop that trend. This lead to a long pause in Bond films until the franchise was resurrected in 1995 with Pierce Brosnan. Also interesting is the fact that with Brosnan, Bond film budgets noticeably increased in size. Only Moonraker comes close to being as expensive as the later Bonds, and that was set in space. 

The early Connery Bond films were the most profitable -- easily topping the current Craig films, by up to a factor of 22 (*Dr. No* yielded 64 times its costs, versus *Quantum of Solace* which made 2.8). Given the quality of Spectre, I fully expect the current Bond to not be a big success (ratings are already very bad in general). This would lead to a new Bond coming up, if history repeats itself -- and Craig has already said he no longer wants to play Bond. 


Code and data for this analysis is availabe on [github](https://github.com/safferli/james_bond_films), as always. 


<!--

Bond: “My dear girl, there are some things that just aren’t done. Such as, drinking Dom Perignon ’53 above the temperature of 38 degrees Fahrenheit. That’s just as bad as listening to the Beatles without earmuffs.” (Goldfinger)

Natalya Simonova: “Do you destroy every vehicle you get into?”
Bond: “Standard operating procedure. Boys with toys.” (Goldeneye)

Bond: “Why is it that people who can’t take advice always insist on giving it?” (Casino Royale)

-->

