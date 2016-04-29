---
layout: post
title:  "Dashboards and Alerts"
date:   2016-04-28 08:07:11
categories: update prometheus monitoring dashboard alert
---

Recap
-----
In the previous post about instrumentation, we saw how to add Prometheus metrics
to our toy webserver application to gain insight into its behavior and state 
(also known as white box monitoring). We exported two types of metrics, a 
counter to track the total count of responses issued by the server, and a distribution to keep tabs on the latency of those responses. Each metric also 
had metric fields which recorded the HTTP response code issued for the request.

Getting insight
---------------
Now that we've got the data, we need a way to use it. The most important aspect
of running any kind of system is understanding what it's doing, which means
monitoring the metrics that let you examine its state. If you don't know what
the current state of your system is, how will you know when your users are
having problems? What if it's just a specific subset of users who are 
experiencing issues? How will you differentiate what is broken for whom? How
will you even know the system is up and running?

Mikey Dickerson (the Site Reliability Engineer who spearheaded the rescue of
HealthCare.gov) [had this] to say about the importance of monitoring:

*"One was that there was no monitoring of the production system. For those of 
you that run large distributed systems, you will understand that this is as if 
you are driving a bus with the windshield covered."*

This is where the use of graphs comes in.

Prometheus has some simple graphing abilities baked into its `/graph`
endpoint, which are useful when you want to create an on-the-fly query or dive 
into a visualization for debugging. Prometheus isn't a graphing application 
though, and it lacks a few important visualization features. For example, we 
might want to create a single dashboard for JVM metrics made up of many 
different graphs in order to examine things such the rate of full GC, the oldgen
or newgen duration in milliseconds, the eden size, or some other Java 
information we're interested in. Prometheus lacks the ability to group these 
various related graphs and save them into a common dashboard, which is why we'll
be installing [Grafana].

Grafana
-------
Grafana is a data visualization application that can produce customizable graphs
and dashboards from input sources you can specify. Unsurprisingly, one of these
sources happens to be Prometheus.

Installing and configuring Grafana
----------------------------------
Since I'm running Ubuntu on my test machine, I followed [these instructions] to
install Grafana. Once installed, first fire up Prometheus so you have a data
source, start the webserver and request generator, then start the Grafana 
service:

{% highlight bash %}
$ ./http_server -port=7070
# ./curler.sh
$ ./prometheus -config.file=/where/your/yaml/lives/prometheus.yaml \
  -storage.local.path=/tmp/data
$ sudo service grafana-server start
{% endhighlight %}

By default, the Grafana service exposes its interface at port 3000. One of the
first things I did was change the password (localhost:3000/admin/users), and
had a look at the config file, which lives at `/etc/grafana/grafana.ini`.

The next step is to actually point Grafana at the Prometheus data being
collected. This can be done with ease by following the instructions for [data
source configuration] on the Grafana website. You should also play around with
[creating dashboards]. For instance, I created an HTTP dashboard with qps, 400s,
and latency statistics.

![http dashboard](http://i.imgur.com/BiKJzEW.png)

That's pretty much all there is to creating dashboards. Grafana has plenty of
other features as well to help you visualize your Prometheus data.

Alerting
--------
Now that we've got dashboards working, it would be nice to not have to keep a
weather eye on them all the time in order to detect when something goes wrong. 
That's what we have alerting for: when a bad situation is detected, we should 
get notified automatically, then use the dashboards and graphs to dig into the
problem. 

Prometheus uses 'alerting rules' to define alert conditions, then sends
notifications to an external service (they recommend [Alertmanager]) to have
these notifications actually routed to an end user (either via email or 
something like [PagerDuty]).

Prometheus alerting rules
-------------------------
Alerting rules are configured in Prometheus in the same way as 
[recording rules]. Alerting rules have a name, expression, duration, labels to
identify the alert (like the severity) and annotations to add additional, non-
identifying data to an alert (such as a playbook link). Consider the following
example, saved as alerting.rules:

```
ALERT High500s
  IF sum(rate(mocking_production_http_server_http_responses_total{code=~"^5..$"}[5m])) / sum(rate(mocking_production_http_server_http_responses_total[5m])) > .002
  FOR 10m
  LABELS { severity = "page" }
  ANNOTATIONS {
    summary = "High 500 ratio on {{ $labels.instance }}.",
    description = "{{ $labels.instance }} has an error ratio of {{ $value }} for 10m.",
    runbook = "www.playbook.com/high500s",
  }
```
This is a rule used to generate an alert for a high rate of 500 responses. In
plain english, it can be interpreted as, 'alert if the rate of 500 responses
divided by the rate of total responses is greater than .2% for 10 minutes'. The
severity of this alert is given as page, and there are helpful annotations which
provide a summary, description, and playbook link for the alert recipient to 
use.

Note that Prometheus provides a helpful tool to check the syntax of an alert
rule, called Promtool. Usage is as follows:

{% highlight bash %}
$ /opt/prometheus-0.17.0.linux-386/promtool check-rules /path/to/alerting.rules
{% endhighlight %}

Once you've got the alerting rules written and validated, you've got to modify
the Prometheus config file to point at the rules file containing the alerts.
Edit prometheus.yml and add the rules filepath to the rule_files section:

{% highlight yaml %}
rule_files:
  # specify aggregations
  - 'prometheus.rules'
  # specify alerting rules
  - '../alerting/alerting.rules'
{% endhighlight %}

If you are running Prometheus already, restart it and check out 
`localhost:9090/alerts`. You should see the rules you've configured show up 
there (hopefully not active, yet). For the http server, I've set up the 500's 
alert I mentioned previously, as well as a high 95th percentile latency 
condition.

![alert not firing](http://i.imgur.com/HmtjeK4.png)

In the last post, we created an endpoint that generated client errors (4xx), but
didn't have anything to generate server errors (5xx) on demand. To create better
dashboards and alerts, I decided to create a simple error handler endpoint,
exposed at "/cause_500". I also wrote another small bash script to send 
requests only to that endpoint.

{% highlight go% }
func errorHandler(writer http.ResponseWriter, request *http.Request) {
    // generate errors on demand
    writer.WriteHeader(http.StatusInternalServerError)
    fmt.Fprintf(writer, "Intentionally caused a 500 error.")
}
...
// added the handler for the endpoint in the main method
http.HandleFunc("/cause_500", errorHandler)
{% endhighlight% }

Armed with these modifications, we can generate alerts and graphs for 500's,
which more realistically mimics an actual production environment, and get the
most out of the High500s alerting rule defined earlier.

Let's look at what the High500s Prometheus alert rule does when we start to
throw some 500's at it. First, fire up the error-generating curler and send
requests to `localhost:7070/cause_500`. When the alert threshold is crossed (in
this case, the ratio of 500's is greater than .002) but the trigger duration 
hasn't been met yet (10 minutes for this alert), the alert rule is considered to
be in the 'pending' state.

![pending alert](http://i.imgur.com/VS0hxET.png)

Once all the alert conditions are satisfied however, the Prometheus alert is
considered active:

![firing alert](http://i.imgur.com/3Qp0jPZ.png)

You can also check out the increase in errors on the Grafana http dashboard:

![error traffic grafana](http://i.imgur.com/iMcpKz2.png)

Installing and configuring Alertmanager
---------------------------------------
Now Prometheus knows when an alert is firing, but we'd like for the oncall for
the service to get notified as well, without having to check the Prometheus
alerting endpoint. This is where Alertmanager comes in. When you run them 
together, Prometheus will send alerts to Alertmanager, who in turn can send 
notifications containing relevant information about the alert to the appropriate
place (such as the pager of the oncall). Alertmanager also takes care of things 
like silencing alerts, or aggregating them together.

Follow the instructions on the [Alertmanager Github page]. You can also use the 
example configuration provided there as a starting place for your own alerting 
config.

Once you've got it installed and have the configuration file setup as you like, 
you can start alertmanager and tell Prometheus where to send the alerts by 
issuing these commands:

{% highlight bash %}
$ ./alertmanager -config.file=/path/to/alertmanager/config/alertmanager_config.yml

$ /opt/prometheus-0.17.0.linux-386/prometheus -config.file=/path/to/prometheus/config/prometheus.yml -storage.local.path=/tmp/data -alertmanager.url=http://localhost:9093/ 
{% endhighlight %}

To test that Alertmanager is receiving alerts properly from Prometheus, fire up
the error generating curl script and hammer the `/cause_500` endpoint for a but.
An active Alertmanager alert will look something like this:

![active alertmanager](http://i.imgur.com/SzhgEUD.png)

Conclusions
-----------
Hopefully you've now got an idea of how to set up alerting in Prometheus and 
Alertmanager, as well as create dashboards in Grafana. Alerting and monitoring
are two of the most important aspects of running a production service, and 
careful thought should be put into the data you graph and the alerts you decide 
to create. These topics are so important that I'll likely be dedicating a future
post to alerting philosphy, and graphing basics.


[Grafana]: http://grafana.org/
[these instructions]: http://docs.grafana.org/installation/debian/
[data source configuration]: http://prometheus.io/docs/visualization/grafana/
[creating dashboards]: http://docs.grafana.org/guides/gettingstarted/
[Alertmanager]: https://github.com/prometheus/alertmanager
[PagerDuty]: https://www.pagerduty.com/
[recording rules]: https://prometheus.io/docs/querying/rules/
[Alertmanager Github page]: https://github.com/prometheus/alertmanager
[had this]: https://fcw.com/articles/2015/03/27/dickerson-at-sxsw.aspx
