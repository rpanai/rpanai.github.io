---
layout: post
title: Earthquakes Data Analisys with Python
---
Scrape tables with <strong>BeautifulSoup</strong>. Create a <strong>Pandas</strong> dataframe and analyze it.

After the last earthquake in [Italy](http://www.usgs.gov/news/magnitude-62-earthquake-central-italy) I was surprised by this [statement](http://www.huffingtonpost.it/2016/08/24/mario-tozzi_n_11672740.html): "Italy is like Middle East. An earthquake of magnitude 6 should not generate such a disaster". There are several things I don't like about that statement, furthermore I moved to New Zealand few months after the [2011 Christchurch earthquake](https://en.wikipedia.org/wiki/2011_Christchurch_earthquake) and that was another magnitude 6 earthquake.

I would like to  analyze the number of deaths for eartquakes of magnitude included between 6 and 6.5. Unfortunatly [USGS](http://www.usgs.gov/) doesn't provide this information so I have to take it from [Wikipedia](https://en.wikipedia.org/wiki/List_of_21st-century_earthquakes).

# 1. Scrape table(s) from Wikipedia

On this [page](https://en.wikipedia.org/wiki/List_of_21st-century_earthquakes) there are the tables reporting all eartquakes from 2001 to date. They are consistently formatted, they report deaths and missings. In the comment columns we can observe that different magnitudos are used but, in order to don't [overcomplicate](http://gji.oxfordjournals.org/content/199/2/805.abstract) things, we can omit it.

The following modules are required

{% highlight python %}
from bs4 import BeautifulSoup
import requests
import pandas as pd
{% endhighlight %}


We need requests to fetch the url and BeautifulSoup to tranform the ugly mess in <code>req.text</code> in somethin more 
"digestible" like a <code>soup</code>.


{% highlight python %}
url="https://en.wikipedia.org/wiki/List_of_21st-century_earthquakes"
headers = {"User-Agent": "Mozilla/5.0"}
req=requests.get(url,headers=headers)
soup = BeautifulSoup(req.text, "lxml")  #Without "lxml" there is a Warning.
{% endhighlight %}

At this point we have to open the [url](https://en.wikipedia.org/wiki/List_of_21st-century_earthquakes) and then view the source code (with right-click or CTRL+U) to understand how tables are html formatted. There is a table for year and it is sortable:

{% highlight html %}
<h2><span class="mw-headline" id="2001">2001</span><span class="mw-editsection"><span class="mw-editsection-bracket">[</span><a href="/w/index.php?title=List_of_21st-century_earthquakes&amp;action=edit&amp;section=1" title="Edit section: 2001">edit</a><span class="mw-editsection-bracket">]</span></span></h2>
<div role="note" class="hatnote">Main article: 
<a href="/wiki/List_of_earthquakes_in_2001" title="List of earthquakes in 2001">
List of earthquakes in 2001</a>
</div>
<table class="wikitable sortable">
{% endhighlight %}

We want to <code>findALL</code> the tables which class is  <code>"wikitable sortable"</code>. Next we should know that the rows in a html table are delimeted by <code>tr</code> so again we want to <code>findALL</code> of them.

{% highlight html %}
<tr>
<td>January 1, 2001</td>
<td>06:57</td>
<td><a href="/wiki/Mindanao" title="Mindanao">Mindanao</a>, Philippines</td>
<td>6.898</td>
<td>126.579</td>
<td style="text-align:right;">0</td>
<td style="text-align:right;">7.5</td>
<td>M<sub>w</sub> (HRV).</td>
<td></td>
</tr>
{% endhighlight %}

Finally every cell is delimited by <code>td</code>, another job for <code>findALL</code>. The last part involve select the cells we want to store and count the cells (they are 9). In this case we want Date and Time (we better merge directly), Latitude, Longitude, Fatalities and Magnitude. These are the columns 0,1,3,4,5,6.

{% highlight python %}
Date=[]
Lat=[]
Long=[]
Death=[]
Mag=[]
for table in soup.findAll(("table", { "class" : "wikitable sortable" })):
    for row in table.findAll("tr"):
        cells= row.findAll("td")
        if len(cells)==9:
            Date.append(cells[0].find(text=True)+' '+cells[1].find(text=True))
            Lat.append(cells[3].find(text=True))
            Long.append(cells[4].find(text=True))
            Death.append(cells[5].find(text=True))  #Only deaths no missings
            Mag.append(cells[6].find(text=True))
{% endhighlight %}

Now we can put all our lists in a dictionary and then create a <code>pandas.DataFrame</code>. 

{% highlight python %}
columns=['Date','Lat','Long','Death','Mag']
diz=dict(zip(columns,[Date,Lat,Long,Death,Mag]))
eqs=pd.DataFrame(diz,columns=columns)
{% endhighlight %}

# 2. Clean the data

The dataframe we just build looks like

{% highlight python %}
In [5]: eqs.head()
Out[5]:

Date	Lat	Long	Death	Mag
0	January 1, 2001 06:57	6.898	126.579	0	7.5
1	January 9, 2001 16:49	−14.928	167.170	0	7.1
2	January 10, 2001 16:02	57.078	−153.211	0	7.0
3	January 13, 2001 17:33	13.049	−88.660	944	7.7
4	January 26, 2001 03:16	23.419	70.232	20,085	7.7

{% endhighlight %}


# 3. Data Analisys

# 4. Plot


<!--

<strong>Bold</strong>
<em>Italics</em>
<u>Underline</u>
![_config.yml]({{ site.baseurl }}/images/config.png)

The easiest way to make your first post is to edit this one. Go into /_posts/ and update the Hello World markdown file. For more instructions head over to the [Jekyll Now repository](https://github.com/barryclark/jekyll-now) on GitHub.
-->
