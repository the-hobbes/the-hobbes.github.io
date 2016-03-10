---
layout: post
title:  "An instrumented webserver in Go"
date:   2016-03-10 15:01:00
categories: update production monitoring golang webserver
---
The first step in this process of mocking production is to create something to monitor. I'll start with the web server, and I've picked Go as the language to implement it in.

As stated in my previous post, the server should respond to both GET and POST requests. We want to have some variety in the responses, so we should plan for the following behaviors in the request handlers:

* POST/GET requests for '/', the site root, should result in 200 OK responses
* POST/GET requests for '/redirect_me' should result in a 302 redirect response
* POST/GET requests for anything else should result in a 400 not found response

These three types of responses should give us enough diversity in monitoring to get us started.

- talk about the basic server in go
- talk about wrapping the request handlers for instrumentation
- talk about what we want to track, and the structure that might take


External instrumentation solutions
===
* https://github.com/n1trux/awesome-sysadmin#monitoring
* https://www.reddit.com/r/golang/comments/439jb4/is_there_a_library_that_can_measure_latency_of/
* https://prometheus.io/
* https://www.reddit.com/r/golang/comments/2ag28l/golang_apps_profiling_and_monitoring_any_tips/
* datadog
* signalfx
