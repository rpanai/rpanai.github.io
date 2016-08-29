---
layout: post
title: Earthquakes and Death Toll with Python
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
Fatalities=[]
Mag=[]
for table in soup.findAll(("table", { "class" : "wikitable sortable" })):
    for row in table.findAll("tr"):
        cells= row.findAll("td")
        if len(cells)==9:
            Date.append(cells[0].find(text=True)+' '+cells[1].find(text=True))
            Lat.append(cells[3].find(text=True))
            Long.append(cells[4].find(text=True))
            Fatalities.append(cells[5].find(text=True))  #Only deaths no missings
            Mag.append(cells[6].find(text=True))
{% endhighlight %}

Now we can put all our lists in a dictionary and then create a <code>pandas.DataFrame</code>. 

{% highlight python %}
columns=["Date","Lat","Long","Fatalities","Mag"]
diz=dict(zip(columns,[Date,Lat,Long,Fatalities,Mag]))
eqs=pd.DataFrame(diz,columns=columns)
{% endhighlight %}

# 2. Clean the data

The dataframe we just build looks like

| | <strong>Date</strong>|<strong>Lat</strong>	 |<strong>Long</strong>	   |<strong>Fatalities</strong>	|<strong>Mag</strong>|
| ----:| ---------------------------- | ----------:| ----------:| ----------:|:------:|
| 0 | January 1, 2001 06:57	|6.898	 |126.579  |0	    |7.5|
| 1 | January 9, 2001 16:49	|−14.928 |167.170  |0	    |7.1|
| 2 |	January 10, 2001 16:02	|57.078	 |−153.211 |0	    |7.0|
| 3 |	January 13, 2001 17:33	|13.049	 |−88.660  |944	    |7.7|
| 4 |	January 26, 2001 03:16	|23.419	 |70.232   |20,085	|7.7|
{:.mbtablestyle}

We observe that all columns are objetcs and we would like to have <code>datetime,float,float,int,float</code> respectively
{% highlight python %}
In [6]:eqs.info()
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 496 entries, 0 to 495
Data columns (total 5 columns):
Date          496 non-null object
Lat           496 non-null object
Long          496 non-null object
Fatalities    496 non-null object
Mag           496 non-null object
dtypes: object(5)
memory usage: 19.5+ KB
{% endhighlight %}

We want to covert the dates in yyyy-mm-dd HH:MM:SS format. The columns Lat and Long have a dash or a hypen instead of a minus. The column Fatalities has some entries with comma as thousand separator <code>20,085</code>, others look like <code>74 dead</code> (we skipped the missing in the scraping process) and finally some ranges <code>87,000~100,000</code>. So we want to remove commas, the word dead and (arbitrarily) we take just the smallest number in the range.  

{% highlight python %}
eqs["Lat"]=eqs["Lat"].apply(lambda x: x.replace('−','-').replace('–','-'))   #replace dash or hypen with minus
eqs["Long"]=eqs["Long"].apply(lambda x: x.replace('−','-').replace('–','-')) #replace dash or hypen with minus
eqs["Fatalities"]=eqs["Fatalities"].apply(lambda x: x.replace(',','').replace(' dead','')) # remove commas and dead
eqs["Fatalities"]=eqs["Fatalities"].apply(lambda x: x.split('~',1)[0]) #In case of a range like 10~15 return the smallest value
#Now we convert Date to datetime and the other columns to numeric
eqs['Date']=pd.to_datetime(eqs['Date']) 
eqs[['Lat','Long','Fatalities','Mag']]=eqs[['Lat','Long','Fatalities','Mag']].apply(pd.to_numeric)
{% endhighlight %}

Everything went right
{% highlight python %}
In [8]:
eqs.info()
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 496 entries, 0 to 495
Data columns (total 5 columns):
Date          496 non-null datetime64[ns]
Lat           496 non-null float64
Long          496 non-null float64
Fatalities    496 non-null int64
Mag           496 non-null float64
dtypes: datetime64[ns](1), float64(3), int64(1)
memory usage: 19.5 KB
{% endhighlight %}

The dataframe looks like

| | <strong>Date</strong>|<strong>Lat</strong>	 |<strong>Long</strong>	   |<strong>Fatalities</strong>	|<strong>Mag</strong>|
| ----:| ---------------------------- | ----------:| ----------:| ----------:|:------:|
|0|	2001-01-01 06:57:00|	6.898|	126.579|	0|	7.5|
|1|	2001-01-09 16:49:00|	-14.928|	167.170|	0|	7.1|
|2|	2001-01-10 16:02:00|	57.078|	-153.211|	0|	7.0|
|3|	2001-01-13 17:33:00|	13.049|	-88.660	|944|	7.7|
|4|	2001-01-26 03:16:00|	23.419|	70.232|	20085|	7.7|
{:.mbtablestyle}

We can save it to csv
{% highlight python %}
eqs.to_csv('earthquakes_from_2001_to_date.csv',index=False)
{% endhighlight %}


# 3. Data Analisys

With the our data we want to plot the earthquakes and their death tolls. There are nice tutorials to plot maps in python: [1](http://introtopython.org/visualization_earthquakes.html), [2](https://www.pfenninger.org/posts/mapping-the-worlds-nuclear-power-plants). We need the following modules

{% highlight python %}
from mpl_toolkits.basemap import Basemap
import matplotlib.pyplot as plt
import pandas as pd
import math
{% endhighlight %}

If we are on a new session we first load the data  

{% highlight python %}
eqs=pd.read_csv("earthquakes_from_2001_to_date.csv")
{% endhighlight %}

{% highlight python %}
df=eqs[eqs.Mag>=6]
m = Basemap(projection='cyl', ellps='WGS84',
            llcrnrlon=-180, llcrnrlat=-90, urcrnrlon=180, urcrnrlat=90,
            resolution='c', suppress_ticks=True)
fig = plt.figure(figsize=(16, 16))
m.drawmapboundary(fill_color=None, linewidth=0)
m.drawcoastlines(color='#4C4C4C', linewidth=0.5)
m.drawcountries()
m.fillcontinents(color='#F2E6DB',lake_color='#DDF2FD')
for index,d in df.iterrows():
    x,y = m(d.Long, d.Lat)
    m.plot(x, y, 'ro',alpha=0.8, markersize=3.0)
title_string = "Earthquakes of Magnitude $M \geq$ 6.0\n"
title_string += "From 2001 to date"
plt.title(title_string)
plt.show()
{% endhighlight %}
![_config.yml]({{ site.baseurl }}/images/2016-08-26-mag_6plus.png)

{% highlight python %}
df=eqs[(eqs.Mag>=6) & (eqs.Mag<=6.5) & (eqs.Fatalities>10)]
m = Basemap(projection='cyl', ellps='WGS84',
            llcrnrlon=-180, llcrnrlat=-90, urcrnrlon=180, urcrnrlat=90,
            resolution='c', suppress_ticks=True)
fig = plt.figure(figsize=(16, 16))
m.drawmapboundary(fill_color=None, linewidth=0)
m.drawcoastlines(color='#4C4C4C', linewidth=0.5)
m.drawcountries()
m.fillcontinents(color='#F2E6DB',lake_color='#DDF2FD')
for index,d in df.iterrows():
    x,y = m(d.Long, d.Lat)
    msize = d.Fatalities/50
    m.plot(x, y, 'ro',alpha=0.8, markersize=msize)
title_string = "Fatalities \n Earthquakes of Magnitude $6.0\leq M\leq 6.5$\n"
title_string += "From 2001 to date"
plt.title(title_string)
plt.show()
{% endhighlight %}
![_config.yml]({{ site.baseurl }}/images/2016-08-26-fatalities.png)

Now it is the time to analyze the data. We restrict the range of magnitude between 6.0 and 6.5 (included),

{% highlight python %}
eqs_range=eqs[(eqs.Mag>=6) & (eqs.Mag<=6.5)]
eqs_range.reset_index(drop=True,inplace=True)
eqs_range.index=eqs_range.index +1
eqs_range[:20]
{% endhighlight %}

| | <strong>Date</strong>|<strong>Lat</strong>	 |<strong>Long</strong>	   |<strong>Fatalities</strong>	|<strong>Mag</strong>|
| ----:| ---------------------------- | ----------:| ----------:| ----------:|:------:|
|1|	2006-05-26 22:53:00|	-7.962|	110.458|	5782|	6.3|
|2|	2002-03-25 14:56:00|	36.062|	69.315|	1000|	6.1|
|3|	2004-02-24 02:27:00|	35.142|	-3.997|	631	|6.4|
|4|	2014-08-03 08:30:00	|27.189	|103.409	|617	|6.2|
|5	|2005-02-22 02:25:00|	30.741|	56.877|	612|	6.4|
|6	|2012-08-11 12:23:00|	38.322|	46.888|	306|	6.4|
|7|	2009-04-06 01:32:00|	42.334|	13.334|	295|	6.3|
|8|	2016-08-24 01:36:00|	42.714|	13.172|	294	|6.2|
|9|	2002-06-22 02:58:00|	35.626|	49.047|	261|	6.5|
|10|	2003-02-24 02:03:00|	39.610|	77.230|	261|	6.3|
|11|	2008-10-28 23:09:00|	30.656|	67.361|	215|	6.4|
|12	|2011-02-21 23:51:00|	-43.583|	172.680|	185|	6.1|
|13|	2003-05-01 00:27:00|	39.007|	40.464|	177|	6.4|
|14|	2016-02-05 19:57:00|	22.939|	120.593|	117|	6.4|
|15|	2006-03-31 01:17:00|	33.581|	48.794|	70	|6.1|
|16	|2007-03-06 03:49:00	|-0.512	|100.524	|67	|6.4|
|17|	2002-02-03 07:11:00|	38.573|	31.271|	44|	6.5|
|18|	2008-08-30 08:30:00|	26.241|	101.889|	43|	6.0|
|19|	2010-03-08 02:32:00|	38.873|	39.981|	42|	6.1|
|20|	2013-04-09 11:52:00|	28.500|	51.591|	37|	6.4|
{:.mbtablestyle}

And do a little of statistic
{% highlight python %}
In [15]:eqs_range.shape[0] #Count entries
Out[15]:84
{% endhighlight %}

{% highlight python %}
In [24]:eqs_range.Fatalities.mean() # Average deaths
Out[24]:135.85714285714286
{% endhighlight %}

And calculate the probability of n or more deaths
{% highlight python %}
prob=lambda x: eqs_range[eqs_range.Fatalities>=x].shape[0]/eqs_range.shape[0]*100
ind=[1,5,10,50,100,200,250,500]
for i in ind:
    print("Deaths \u2265 {:4d}, Prob: {:7.2f}%".format(i,prob(i)))
    
Deaths ≥    1, Prob:  100.00%
Deaths ≥    5, Prob:   47.62%
Deaths ≥   10, Prob:   38.10%
Deaths ≥   50, Prob:   19.05%
Deaths ≥  100, Prob:   16.67%
Deaths ≥  200, Prob:   13.10%
Deaths ≥  250, Prob:   11.90%
Deaths ≥  500, Prob:    5.95%
{% endhighlight %}
This says that every earthquake of magnitude between 6.0 and 6.5 in the XXI century comes with id death toll but just sligthly less than 40% have a death toll of 10 or more. In this setting
{% highlight python %}
In [51]:eqs_range[eqs_range.Fatalities>=10]['Fatalities'].mean()
Out[51]:352.59375
{% endhighlight %}
{% highlight python %}
prob2=lambda x: eqs_range[(eqs_range.Fatalities>=10)  & 
                          (eqs_range.Fatalities>=x)].shape[0]/eqs_range[eqs_range.Fatalities>=10].shape[0]*100
for i in ind[3:]:
    print("Deaths \u2265 {:4d}, Prob: {:7.2f}%".format(i,prob2(i)))

Deaths ≥   50, Prob:   50.00%
Deaths ≥  100, Prob:   43.75%
Deaths ≥  200, Prob:   34.38%
Deaths ≥  250, Prob:   31.25%
Deaths ≥  500, Prob:   15.62%
{% endhighlight %}


# 4. Conclusions(?)
In order to draw conclusions we need far more data as: deep and duration of earthquakes, the distance from epicentre to first town  and/or the density of population at any given point. Right now we can say that eartquakes of magnitude 6.0~6.5 always kill people and if they kill more than 10 person the chance they kill more than 200 is one over three.

<!--
{% highlight python %}
{% endhighlight %}

<code></code>

<strong>Bold</strong>
<em>Italics</em>
<u>Underline</u>
![_config.yml]({{ site.baseurl }}/images/config.png)

The easiest way to make your first post is to edit this one. Go into /_posts/ and update the Hello World markdown file. For more instructions head over to the [Jekyll Now repository](https://github.com/barryclark/jekyll-now) on GitHub.
-->
