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

Tracking these three types of responses should give us enough diversity in monitoring to get us started. With that in mind, lets have a look at about the simplest web server you can make in Go:

```
package main

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

The above code is taken from the [Writing Web Applications](https://golang.org/doc/articles/wiki/#tmp_3), tutorial, and functions basically as follows.

In the `main` function, we set up a handler for all requests to the root of the site ('/'). The call to [`HandleFunc`](https://golang.org/pkg/net/http/#HandleFunc) registers the handler function you provide for a pattern (in this `handler` is registered for case '/') in the DefaultServeMux, discussed below. You can also use the [Handle](https://golang.org/pkg/net/http/#Handle) function to register handlers.

The next line is a call to ListenAndServe, which is what actually starts the HTTP server. It accepts two arguments, a port and a handler of type [Handler](https://golang.org/pkg/net/http/#Handler). By providing `nil` as the handler, ListenAndServe defaults to DefaultServeMux, which is an instance of the [ServeMux](https://golang.org/pkg/net/http/#ServeMux) type, an HTTP request multiplexer. From the docs, 'ListenAndServe listens on the TCP network address addr and then calls Serve with handler to handle requests on incoming connections.'

As for the `handler` function itself, it takes a ResponseWriter and Request object and uses [Fprintf](https://golang.org/pkg/fmt/#Fprintf) to write a message to the ResponseWriter that includes the request path (slicing off the preceeding '/' with the `[1:]` syntax).

Now we've got a basic HTTP server. If we save it as http.go and have the 'go' command line tool installed, we can start it up from the command line using `go run http.go`. You can either visit `localhost:8080/` in your browser, or run a command like `curl -X GET localhost:8080/` to send requests to the server.

The next step is to add some instrumentation to the server. To do this, we'll want to wrap the handler in such a way that for every request it processes, we collect some interesting stats. Possibilities include:

* the total number of requests
* the total number of errors returned by the server
* the latency for each request

Given the nature of and HTTP response, it would be useful to also add some metadata to each of these stats to track a breakdown of response by class. For example, the total number of 200 OK requests, or the total number of 500 errors. I'll get into some of the implementation details of this in the next post, such as the right format for these metrics (eg a distribution for latency stats). We also might want to add some logging, so we can see each request as it is processed by the server.

To demonstrate how one might perform the handler wrapping, lets take a look at some more code.

```
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"
)

type WrapHTTPHandler struct {
	handler http.Handler
}

type LoggedResponse struct {
	http.ResponseWriter
	status int
}

func (wrappedHandler *WrapHTTPHandler) ServeHTTP(writer http.ResponseWriter, request *http.Request) {
	loggedWriter := &LoggedResponse{ResponseWriter: writer, status: 200}
	start := time.Now()
	wrappedHandler.handler.ServeHTTP(loggedWriter, request)
	elapsed := time.Since(start)
	log.SetPrefix("[Info]")
	log.Printf("[%s] %s - %d, time elapsed was: %dns.\n",
		request.RemoteAddr, request.URL, loggedWriter.status, elapsed)
}

func (loggedResponse *LoggedResponse) WriteHeader(status int) {
	loggedResponse.status = status
	loggedResponse.ResponseWriter.WriteHeader(status)
}

func rootHandler(writer http.ResponseWriter, request *http.Request) {
	// The "/" pattern matches everything, so we need to check that we're at the root here.
	if request.URL.Path != "/" {
		http.NotFound(writer, request)
		return
	}
	fmt.Fprintf(writer, "You've hit the home page.")
}

func main() {
	http.HandleFunc("/", rootHandler)
	http.Handle("/redirect_me", http.RedirectHandler("/", http.StatusFound))
	http.ListenAndServe(":8080", &WrapHTTPHandler{http.DefaultServeMux})
}
```

We'll begin with what is happening in the main function. First, we use HandleFunc to register a handler function for the site root, called rootHandler. We then use Handle to register a built in RedirectHandler for the path "/redirect_me", which we'll use to generate 302 responses via the StatusFound directive. With these two handler's registered, we'll actually start the server as we did in the previous example, using ListenAndServe. The key difference this time is the second argument to ListenAndServe, which happens to be `&WrapHTTPHandler{http.DefaultServeMux}` instead of nil.

The `WrapHTTPHandler` type serves as a wrapper around the Handler for ListenAndServe, allowing us to execute our own code each time a request is handled, along with the regular functionality that the DefaultServeMux provides. Since a Handler is defined as a type of object that implements the [ServeHTTP](https://golang.org/pkg/net/http/#HandlerFunc.ServeHTTP) method, that is what we need to override to add our metric code.

Indeed, when we look at our implementation of ServeHTTP, we can see we've added a few things, before and after the call to the original handler.ServeHTTP function which actually performs the serving transaction. First, we create a new instance of `LoggedResponse`, a type we've created to add some additional information in the form of the HTTP response code (`status`)to the builtin [ResponseWriter](https://golang.org/pkg/net/http/#ResponseWriter) type used to send HTTP replies. Next, we start a timer and kick off the call to `wrappedHandler.handler.ServeHTTP`, passing our loggedWriter instance and request to be served by the default ServeHTTP method we get from the handler. We then take the elapsed time (to be used later in our latency measurements), and set up some logging before writing all of that information out.

This means that whenever a user makes a request for the site root and `rootHandler` is called, the wrapped `ServeHTTP` will be used to serve the response. The same goes for the redirect handler, `http.RedirectHandler`. In this way, we can inject some simple metrics into our handling code.

Finally, here's a simple script that you can use to generate some requests to your server and start seeing some responses.

```
#!/bin/bash
for i in `seq 1 50`
do
  curl -X GET localhost:8080/
  curl -X POST localhost:8080/
  curl -X GET localhost:8080/fail
  curl -X GET localhost:8080/redirect_me
done
```

External instrumentation solutions
===
During the course of researching this blog post, I came across some tools that are used to perform metric gathering in the production environments in industry. Here's a list of what I turned up. If I've missed your favorite, or if you experience with any of these tools, please let me know.

* [Profiling Go](http://blog.golang.org/profiling-go-programs) - Official GoLang blog post about profiling go programs
* [python-diamond](https://github.com/python-diamond/Diamond) - A python based statistic collection daemon.
* [Collectd](https://collectd.org/) - A general system statistic collection daemon that looks [like it](https://github.com/collectd/go-collectd) could be used to monitor Go binaries
* [Collectl](http://collectl.sourceforge.net/) - Another system performance metrics collecting tool.
* [Statsd](https://github.com/etsy/statsd) - An application statistic listener developed by etsy.
* [Prometheus](https://prometheus.io/) - An open-source service monitoring system and time series database.
  * It also has a Go [client](https://github.com/prometheus/client_golang) and [transformation server](https://github.com/prometheus/prometheus)
* [go-metrics](https://github.com/rcrowley/go-metrics) - A go port of a [JMV metric collection library](https://github.com/dropwizard/metrics)

Next post I'll delve into the metrics themselves more deeply, and how to structure and store them. I'd also like to explore a few of the above tools in future posts.
