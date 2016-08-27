---
layout: post
title: Earthquakes Data Analisys 
---
Scrape tables with BeautifulSoup, create a Pandas dataframe and analyze it.


After the last earthquake in [Italy](http://www.usgs.gov/news/magnitude-62-earthquake-central-italy) I was surprised by this [statement](http://www.huffingtonpost.it/2016/08/24/mario-tozzi_n_11672740.html): "Italy is like Middle East. An earthquake of magnitude 6 should not generate such a disaster". There are several things I don't like about that statement, furthermore I moved to New Zealand few months after the [2011 Christchurch earthquake](https://en.wikipedia.org/wiki/2011_Christchurch_earthquake) and that was another magnitudo 6 earthquake.

I would like to  analyze the number of deaths for eartquakes of magnitudo included between 6 and 6.5. Unfortunatly [USGS](http://www.usgs.gov/) doesn't provide this information so I have to take it from [Wikipedia](https://en.wikipedia.org/wiki/List_of_21st-century_earthquakes).

# Scrape table(s) from Wikipedia

On this [page](https://en.wikipedia.org/wiki/List_of_21st-century_earthquakes) there are the tables reporting all eartquakes from 2001 to date. They are consistently formatted, they report deaths and missings. In the comment columns we can observe that different magnitudos are used but, in order to don't [overcomplicate](http://gji.oxfordjournals.org/content/199/2/805.abstract) things, we can omit it.

We are going to need the following modules
{% highlight python %}
from bs4 import BeautifulSoup
import requests
import pandas as pd
{% endhighlight %}
<!--
![_config.yml]({{ site.baseurl }}/images/config.png)

The easiest way to make your first post is to edit this one. Go into /_posts/ and update the Hello World markdown file. For more instructions head over to the [Jekyll Now repository](https://github.com/barryclark/jekyll-now) on GitHub.
-->
