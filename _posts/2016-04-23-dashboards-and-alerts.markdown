---
layout: post
title:  "Dashboards and Alerts"
date:   2015-04-23 09:43:11
categories: update prometheus monitoring dashboard alert
---

Recap
-----
In the previous post about instrumentation, we saw how to add Prometheus 
metrics to our toy webserver application to gain insight into its behavior and
state (white box monitoring). We exported two types of metrics, a counter to
track the total count of responses issued by the server, and a distribution to
keep tabs on the latency of those responses. Each metric also had metric fields
which recorded the HTTP response code issued.

Getting insight
---------------
Now that we've got the data, we need a way to use it. The most important aspect
of running any kind of system is understanding what it's doing, which means
monitoring key metrics that let you examine its state. If you don't know what
the current state of your system is, how will you know when your users are
having problems? What if it's just a specific subset of users? How will you
know what is broken for whom? How will you even know the system is up and 
running?

This is where the use of graphs comes in.

Prometheus has some simple graphing abilities baked into its `/graph`
endpoint, but it lacks a few important features. For example, we might want to create a single dashboard for JVM metrics made up of many different graphs in
order to examine things succh the rate of full garbage collections, the oldgen or newgen duration in milliseconds, the eden size, or some other Java metrics.
Prometheus lacks the ability to group these various related graphs and save 
them into a common dashboard, which is why we'll be installing [Grafana].

Grafana
-------

Installing and configuring Grafana
----------------------------------

Alerting
--------

Installing and configuring Alertmanager
---------------------------------------

Putting it all together
-----------------------

Conclusions
-----------


[Grafana]: http://grafana.org/
