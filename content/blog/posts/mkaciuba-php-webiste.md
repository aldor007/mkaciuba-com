+++
title = "PHP mkaciuba Webiste"
date = 2017-04-01T11:42:21+01:00
images = []
tags = ["devops", "www"]
categories = ["projects"]
draft = false
type = "posts"
+++

This project was the longest (except my master degree) I have ever done. The website is composed of few elements and I was trying to learn as much as I can while doing it.

# **Big picture**

![](https://mort.mkaciuba.com/media/blog/14695/89/e43f9e40aeff67b8285d6250c7a3752ada725151.png)![](https://mort.mkaciuba.com/media/blog/0001/01/40ba3b04b28904bc0088ff3b1e3cb7cfaebe2fa5.png)

## **Cloudflare**

As a first line of defence I’m using the Cloudflare. It is a global Content Delivery Network that helps my site to be faster in all corners of the globe. I’m using CDN for cache all static content and it provides DNS very easy to use. Furthermore by using the Cloudflare I’ve got free wildcard certificate. Maybe in times of Let’s Encrypt it isn’t surprising but it is working without any effort from me.

## **NGINX**

As HTTP server I’m using nginx. It is very fast non-blocking HTTP server that gives a lot of opportunities to customize traffic. It can be greatly extended by the use of a little scripting in LUA. I’m using LUA  to check which type is user’s device and check if I should redirect it to HTTPS. I’m still thinking about doing more.

## **PHP**

My application is written in PHP using Symfony framework. Symfony has very mature approach to process request. I’m calling it enterprise PHP. Biggest advantages of Symfony is that it has a very big community that provides great support and libraries for everything.I will provide more information about backend in next section of this post.

## **Databases**

**Redis** - very fast in memory key-value store.  I’m using it as cache for many types of objects. For example: cache for ORM result, for rendered HTML, for block etc.

**MySQL** - most common relational database being used in web services.

**Elastica** (elasticsearch) - document “databases” of my application are using it for full text search across content.

**Beanstalkd**  -  very simple messages queue.

# **Backend application**

While building this application, I was paying attention to make it easy to use and very like some popular CMS. I have one though in my head “All things it should accomplish by using panel”. I’m caching every request to database so when I’m editing, content/gallery application creates purge request that is going to queue, and then worker is purging it from local nginx and Cloudflare. Task queue isn’t used only to purging but I can add images to gallery from a ZIP archive. Worker extracts files and adds it to a database and then process some predefined transformations on images

# **Performance comparison**

I was testing my application with latest docker images of wordpress. Without cache my application is 2x faster than wordpress. So it’s quiet good.  

## No cache - wordpress 

```bash
$ ab  -n 1000 -c 2 http://mkaciuba.com/
Requests per second:    2.50 [#/sec] (mean)
Time per request:       799.586 [ms] (mean)
Time per request:       399.793 [ms] (mean, across all concurrent requests)
Transfer rate:          150.51 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:       37   45   9.3     44     263
Processing:   476  754 346.5    535    2615
Waiting:      392  664 345.5    444    2529
Total:        518  799 346.7    581    2670

Percentage of the requests served within a certain time (ms)
  50%    581
  66%   1030
  75%   1038
  80%   1043
  90%   1075
  95%   1540
  98%   1569
  99%   2050
 100%   2670 (longest request)

$ ab  -n 1000 -c 10 http://mkaciuba.com/

Requests per second:    2.05 [#/sec] (mean)
Time per request:       4877.444 [ms] (mean)
Time per request:       487.744 [ms] (mean, across all concurrent requests)
Transfer rate:          123.37 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:       39   44   4.4     43      83
Processing:   483 4810 2169.0   4534   15492
Waiting:      396 4720 2169.7   4432   15407
Total:        524 4855 2169.6   4577   15536

Percentage of the requests served within a certain time (ms)
  50%   4577
  66%   5535
  75%   6033
  80%   6070
  90%   7534
  95%   8546
  98%  10537
  99%  11601
 100%  15536 (longest request)

```

## Cache Wordpress
```bash
$ ab  -n 1000 -c 10 http://mkaciuba.com/
Requests per second:    34.64 [#/sec] (mean)
Time per request:       288.700 [ms] (mean)
Time per request:       28.870 [ms] (mean, across all concurrent requests)
Transfer rate:          2087.99 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:       39   46  19.0     44     271
Processing:   156  241  45.1    258     514
Waiting:       70  152  45.0    171     422
Total:        198  287  48.3    304     564

Percentage of the requests served within a certain time (ms)
  50%    304
  66%    312
  75%    317
  80%    319
  90%    328
  95%    337
  98%    397
  99%    434
 100%    564 (longest request)

```

## My application no  HTTP cache

```bash
$ ab  -n 1000 -c 2 http://mkaciuba.com/
Requests per second:    3.54 [#/sec] (mean)
Time per request:       564.740 [ms] (mean)
Time per request:       282.370 [ms] (mean, across all concurrent requests)
Transfer rate:          105.59 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:       39   43   4.9     42     120
Processing:   254  521 270.9    294    1771
Waiting:      210  476 270.9    249    1727
Total:        295  564 270.9    339    1813

Percentage of the requests served within a certain time (ms)
  50%    339
  66%    806
  75%    810
  80%    812
  90%    822
  95%    835
  98%   1299
  99%   1310
 100%   1813 (longest request)

$ ab  -n 1000 -c 10 http://wp.mkaciuba.com/
Requests per second:    3.99 [#/sec] (mean)
Time per request:       2507.573 [ms] (mean)
Time per request:       250.757 [ms] (mean, across all concurrent requests)
Transfer rate:          118.91 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:       37   43   4.3     42     112
Processing:   261 2453 1082.0   2282   10274
Waiting:      218 2409 1082.0   2238   10228
Total:        301 2496 1082.2   2324   10314

Percentage of the requests served within a certain time (ms)
  50%   2324
  66%   2822
  75%   3307
  80%   3313
  90%   3805
  95%   3873
  98%   4800
  99%   5809
 100%  10314 (longest request)

```

## My application cache

```bash
$ ab  -n 1000 -c 10 http://mkaciuba.com/
Requests per second:    36.94 [#/sec] (mean)
Time per request:       270.718 [ms] (mean)
Time per request:       27.072 [ms] (mean, across all concurrent requests)
Transfer rate:          1101.92 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:       37   47  51.1     43    1498
Processing:   110  222 112.1    237    1367
Waiting:       67  178 112.0    193    1312
Total:        151  269 122.9    282    1757

Percentage of the requests served within a certain time (ms)
  50%    282
  66%    287
  75%    291
  80%    293
  90%    301
  95%    321
  98%    413
  99%   1215
 100%   1757 (longest request)
 ```