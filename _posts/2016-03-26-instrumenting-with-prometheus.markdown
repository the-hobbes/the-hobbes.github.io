---
layout: post
title:  "Adding instrumentating with Prometheus"
date:   2016-03-26 22:06:47
categories: update prometheus metrics instrumentation monitoring
---
This post will take a look at using the Prometheus monitoring system to set up
instrumentation in the webserver we built last post. I'll start by giving an
overview of Prometheus, and then dive into some code examples.

Prometheus
----------
[Prometheus] is an open source service monitoring system and timeseries
database. It comes with some basic visualization and has a built-in query
language to support detailed examination of exported metrics.

You can get up and running with Prometheus quickly by following the hello world
example on their website. The basic system consists of the Prometheus binary 
itself, a configuration you write in yaml and provide to it, and the
instrumentation you add via one of the many Prometheus [client libraries]. If
there isn't client code for the language your project uses, you can also send
data to the Prometheus server via one of two [exposition formats]. 

In addition to the basics, the Prometheus system also has support for [alerting]
and [visualization]. Graphana [can also display] Prometheus data, which is
something I'd like to explore in a future post.

Configuration
-------------
Settinng up the configuration for Prometheus is relatively simple. The full
options for configuration are [enumerated in the docs], but to get up and
running this is pretty much what you need:

{% highlight yaml %}
global:
  scrape\_interval: 15s
  external\_labels:
    monitor: 'http-example'

rule\_files:
  - 'prometheus.rules'

scrape\_configs:
  - job\_name: 'prometheus'
  target\_groups:
    - targets: ['localhost:9090']

  - job\_name: 'http-server'
  scrape\_interval: 5s
  target\_groups:
    - targets: ['localhost:8080']
{% endhighlight %}

This config file sets up two targets for prometheus to scrape exported metrics
from, one that runs on port 9090 that we label 'prometheus' (the prometheus
binary itself), and one that runs on 8080 that we label 'http\_server', which is
the http server we wrote in the last post. There are also some other 
specifications in the config, like a scrape interval (globally 5 seconds, but
overriden for our http-server target to 5 seconds), and a rule file, in which
you can specify aggregations or precomputations that you'd like to collect in
addition to the raw metrics being exported.

You can run the server using the following flags:

{% highlight bash %}
$ ./prometheus -config.file=/where/your/yaml/lives/prometheus.yaml \
  -storage.local.path=/tmp/data
{% endhighlight %}

If you've used the above config file, Prometheus will already be scraping
itself, and you can view the data being collected by visiitng
localhost:9090/metrics, or checkout the expression browser and graphs by going
to localhost:9090/graph.

Instrumentation
---------------
With Prometheus itself configured and running properly, the next step is to add
code to export the metrics we are interested in tracking. In an http application
like the one we are writing, I mostly want to keep tabs on the request count,
error rate (HTTP 500s), and response latency. If I want to track these things,
I'll first need to choose the right kind of metric to represent them.

The 'Counter' metric is a simple, cumulative count of some value. Counters are
continuously increasing metrics that are useful when paired with a time
interval (eg '... in the last 10 minutes'). Common examples of counters include
requests served, errors served, or job failures. 

'Gauge' is another type of useful metric. It gives the instantateous
representation of a value like the cpu/memory usage on a machine, or the number
of running instances of a given job. These values will still be useful without a
corresponding time interval, which isn't true of Counters.

'Distribution' is another commonly used metric. A distribution 


Putting it together
-------------------

[prometheus]: https://prometheus.io/
[client libraries]: https://prometheus.io/docs/instrumenting/clientlibs/
[exposition formats]: https://prometheus.io/docs/instrumenting/exposition_formats/
[alerting]: https://prometheus.io/docs/alerting/overview/
[visualization]: https://prometheus.io/docs/visualization/browser/
[can also display]: https://prometheus.io/docs/visualization/grafana/
[enumerated in the docs]: https://prometheus.io/docs/operating/configuration/

