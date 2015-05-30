---
layout: post
title:  "Protocol Buffers in Go"
date:   2015-05-30 15:01:00
categories: update protobuf golang
---
In the interest of learning [Go][golang] and more about [Google's Protocol Buffers][protobuf], I followed [this][tutorial] excellent tutorial to build a TCP client and server that would send and receive protobufs, all in Go. 

I won't rehash anything that was covered in the step by step guide. It goes over building a TCP client and server, writing a protocol buffer file and generating the code to work with it, as well as some Go features such as Goroutines. I will say that I had a bit of trouble with the install- I made a hash of it and ended up with the protocol buffer packages in one place, the Go libraries in another, and some very confused $PATH and $GOPATH variables. This [stackoverflow][stack] answer was helpful, and I ended up just munging everything into my $GOPATH once I found where it had all been put. There was a lot of this:

{% highlight bash %}
sudo find / -type d -name "protobuf"
{% endhighlight %}

I would also suggest that you throw functions you want to reuse in a Constants.go file (with the appropriate packagename defined) so the server and client can share them. I also installed the [Go Linter][lint] to run some checks against my formatting. There's also a [cool package][sublimeLint] for Sublime Text that provides an interface to Golint from that editor. 

Working through the tutorial was an instructive experience. My finished client/server [package is here][mystuff], with loads of comments. Nice way to spend a few hours on a Saturday.

[golang]:         http://golang.org/
[tutorial]:       http://www.minaandrawos.com/2014/05/27/practical-guide-protocol-buffers-protobuf-go-golang/
[protobuf]:       https://developers.google.com/protocol-buffers/
[stack]:          http://stackoverflow.com/questions/13214029/go-build-cannot-find-package-even-though-gopath-is-set
[lint]:           https://github.com/golang/lint
[sublimeLint]:    https://packagecontrol.io/packages/SublimeLinter-contrib-golint
[mystuff]:        https://github.com/the-hobbes/LearnGo/tree/master/Protobuff/Tutorial/src