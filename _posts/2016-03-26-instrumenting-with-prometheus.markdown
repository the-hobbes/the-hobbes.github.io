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

'Distribution' is another commonly used metric type. Distributions (or
histograms) group sampled values into different 'bins', and the count of values
in these bins is used to represent the frequency of values that fall into the
range represented by the bin. Distributions are especially useful when used to
represent things like size or duration (request latency or request size).

Distributions can also be used to calculate quantiles and percentiles of their
tracked data, which is comes in handly when you want to see things that might be
hidden from you by using mean values to represent the health of your system.
Take latency for example. If your mean or even 50th percentile (median) request
latency is 80 milliseconds, you might interpret that to mean the system is
running well and user requests are being served in a timely manner. However,
graphing 90th percentile latency could easily paint a different picture- if your
99th percentile latency is 10 seconds, 1% of your users are experiencing request
latency of 10+ seconds, which should probably be investigated.

Given this information, we'll want to instrument our http server using counters
to represent the requests and errors, and a distribution type metric to track
the request latencies. Happily, Prometheus provides all of these metric types in
their [Golang client library].

Using the client library
------------------------
I should note that Prometheus actually comes with its [own http hander] that
exports some http metrics by default. I'm not using it in my example, since I
wanted to get a feel for creating the metrics myself.

To create a counter tracking the HTTP responses, you can use the NewCounterVec
function provided by the Prometheus library.

{% highlight go %}
httpResponsesTotal = prometheus.NewCounterVec(
  prometheus.CounterOpts{
    Namespace: "mocking_production",
        Subsystem: "http_server",
        Name:      "http_responses_total",
        Help:      "The count of http responses issued, classified by code and method.",
    },
    []string{"code", "method"},
)
{% endhighlight %}

In this snippet, I've defined a counter called `httpResponsesTotal`, and added
some contextual information to it by using `prometheus.CounterOpts`. The
namespace, subsystem and name will all become part of the metric name and help provide clarifying details about what the counter does.

I've also annotated the counter with two strings, "code" and "method". These
will be the HTTP response code provided by the http server and the HTTP request method, respectively. These will be added to the counter metric as labels, which means we don't need a separate counter for errors. Instead, we can use the
Prometheus query language to show only those requests with a code of 500 or
greater, like `http_requests_total{code=~"^5..$"}`. I created the distribution
metric in a similar fashion, using `prometheus.NewHistogramVec`.

Putting it together
-------------------

[prometheus]: https://prometheus.io/
[client libraries]: https://prometheus.io/docs/instrumenting/clientlibs/
[exposition formats]: https://prometheus.io/docs/instrumenting/exposition_formats/
[alerting]: https://prometheus.io/docs/alerting/overview/
[visualization]: https://prometheus.io/docs/visualization/browser/
[can also display]: https://prometheus.io/docs/visualization/grafana/
[enumerated in the docs]: https://prometheus.io/docs/operating/configuration/
[Golang client library]: https://github.com/prometheus/client_golang
[own http hander]: https://github.com/prometheus/client_golang/blob/90c15b5efa0dc32a7d259234e02ac9a99e6d3b82/prometheus/http.go#L60
