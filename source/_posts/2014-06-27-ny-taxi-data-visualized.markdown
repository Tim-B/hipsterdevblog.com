---
layout: post
title: "NY Taxi Data Visualized"
date: 2014-06-27 11:58:54 +1000
comments: true
author: Michael B
categories: 
---
Recently a massive dataset of [NYC Taxi Data](http://chriswhong.com/open-data/foil_nyc_taxi/) was made public. There are torrents available but at 19gb the data can be quite unwieldy to manage on a home machine. /r/BigQuery have [uploaded](http://www.reddit.com/r/bigquery/comments/28ialf/173_million_2013_nyc_taxi_rides_shared_on_bigquery/) the dataset to Google's BigQuery service.

BQ provides a simple way to get insights out of this dataset without tearing through your internet usage or waiting for your home machine to query 173 million records. For example on reddit they have already discovered some [anonymization issues](https://medium.com/@vijayp/of-taxis-and-rainbows-f6bc289679a1).

I've taken some of the popular Queries and charted them.

<!-- more -->

##Histogram of tips as a % of fare.


<div>

{% render_partial _partials/histogram_tips.html %}

</div>

{% codeblock lang:sql %}

SELECT INTEGER(ROUND(FLOAT(tip_amount) / FLOAT(fare_amount) * 100)) tip_pct,
  count(*) trips
FROM [833682135931:nyctaxi.trip_fare] 
WHERE payment_type='CRD' and float(fare_amount) > 0.00
GROUP BY 1
ORDER BY 1

{% endcodeblock %}

##Average Speed Over Hour.


<div>

{% render_partial _partials/speed_time.html %}

</div>

{% codeblock lang:sql %}

SELECT
  HOUR(TIMESTAMP(pickup_datetime)) as hour,
  ROUND(AVG(FLOAT(trip_distance)/FLOAT(trip_time_in_secs)*60*60)) AS speed
FROM
  [833682135931:nyctaxi.trip_data]
WHERE
  INTEGER(trip_time_in_secs) > 10
  AND FLOAT(trip_distance) < 90
GROUP BY
  hour
ORDER BY
  hour;

{% endcodeblock %}

##Average Tip Over Month.


<div>

{% render_partial _partials/month_tip.html %}

</div>

{% codeblock lang:sql %}

SELECT INTEGER(AVG(tip_amount)*100)/100 avg_tip,
  REGEXP_EXTRACT(pickup_datetime, "2013-([0-9]*)") month
FROM [833682135931:nyctaxi.trip_fare] 
WHERE payment_type='CRD'
GROUP BY 2
ORDER BY 2

{% endcodeblock %}








