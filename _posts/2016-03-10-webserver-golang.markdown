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
