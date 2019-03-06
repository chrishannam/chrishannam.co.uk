---
layout: post
title:  "StatsD And Anomalies"
date:   2016-08-10 12:16:54 +0000
categories: statsd r
---
# Anomaly Detection
I had been looking for a tool to detect anomalies in data. I stumbled across two libraries from Twitter:

* [Anomaly Detection](https://github.com/twitter/AnomalyDetection)
* [Breakout Detection](https://github.com/twitter/BreakoutDetection)

These are R libraries for analysis of data. I have written a quick script to take data exported from [StatsD](https://github.com/statsd/statsd)
and plot a graph with the interesting parts highlighted.

I was writing R code in a text editor but then someone suggested [RStudio](https://www.rstudio.com/) which I would highly recommend.

Below is the graph I was able to generate with 28 days of data.

![alt text](/assets/2016/08/statsd-plot.png "StatsD Plot.")
Spot the issue…

The circled areas are available in code as well:

```r
> print(res$anoms)
            timestamp anoms
1 2016-08-24 22:25:00 419.4919
2 2016-08-24 22:55:00 546.4654
3 2016-08-24 23:00:00 276.6360
4 2016-08-26 16:15:00 106.3696
```
