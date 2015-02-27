---
layout: post
title: "Geo2TZ - Find the Timezone of a Set of Coordinates"
modified:
categories: ['ruby']
excerpt:
tags: ['geo2tz', 'ruby', 'gem']
image:
  feature:
date: 2013-05-23T22:53:26-08:00
---

At work we really needed a tool to give us a timezone for a specific sets of coordinates but we couldn’t rely on external services since they can be quite unreliable/slow.

This is why I created the geo2tz gem. The idea behind it is quite simple.

Geonames.org provides a list of all cities in the world that have more than 1000 residents. For each city they provide the latitude / longitude as well as the timezone.

Given that we can use a KDTREE implementation to find the closest location of the lat/long we are looking for and return the timezone.

> In computer science, a k-d tree (short for k-dimensional tree) is a space-partitioning data structure for organizing points in a k-dimensional space. k-d trees are a useful data structure for several applications, such as searches involving a multidimensional search key (e.g. range searches and nearest neighbor searches). k-d trees are a special case of binary space partitioning trees.


## Setting it up

gem install geo2tz
In order to work this gem needs to download the city list from geonames, parse it and store it somewhere for future use.
So you at least need to specify a writable directory where we can do all that.

{% highlight ruby %}
Geo2tz.configure(
  :writable_directory => '/myproject/files,
  :filename => 'geo2tz.db',# Optional
  :geoname_url => 'http://download.geonames.org/export/dump/cities1000.zip',# Optional
  :geoname_filename => 'cities1000.txt'# Optional
)
{% endhighlight %}

##Initial Data

Before you can start using the gem you need to fetch the cities from geonames.

You can either use the

{% highlight ruby %}
Geo2tz::Updater.new
{% endhighlight %}

or if you are using rails, you can use the rake task.

{% highlight ruby %}
rake geo2tz::fetch_geodata
{% endhighlight %}

## Usage

{% highlight ruby %}
geo2tz::Search.lat_long(-18,31.5)
=> (GMT+02:00) Africa/Harare
{% endhighlight %}

##ADDITIONAL NOTES

The very first time you use the search function it might be a little long (0.5sec or so) because it populates the tree for the first time. Once it’s loaded it’s extremely fast.

Have fun!