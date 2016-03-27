---
layout: post
title:  "Mocking production"
date:   2016-03-05 15:01:00
categories: update production monitoring
---
I’ve recently decided to undertake a little personal project to create a mock production environment. The goal is to simulate some basic components of a web application and set up monitoring, alerting, and logging for it. The purpose of this toy system is to get me familiar with the tools available in the broader industry today, and facilitate a deeper understanding of the roots of SRE philosophy.

Goals
=====
1. Create a simple but functioning web application that responds to regular HTTP and RPC requests
2. Implement monitoring (including logging) and alerting for this application using industry standard technology and homebrewed solutions where applicable
3. Thoroughly understand what goes into building a solid production environment and maintaining it
4. Learn more about logging, monitoring, alerting, Java, and GoLang

Objectives
==========
1. Create a simple web server in Go that responds to common HTTP GET and POST requests
2. Create a simple api server (also in Go) that responds to RPCs
3. Create simple ‘hammer’ applications in Java that can simulate high volumes of traffic to the Go web and api servers
4. Host these applications on small GCE or AWS servers, depending on what is most cost effective

Specifications
==============
The goal of this project is to increase my monitoring/alerting knowledge, and my understanding of how production environments can work. Therefore I want to keep the components as simple as possible, so I can concentrate on the things I’d like to learn. This will be a toy system, but hopefully one from which I can learn some things.

Hosting
=======
There are a few hosting options available to me where my server and hammer applications can live. I’d like to have them hosted an not run them locally since I’d like to learn more about hosting applications and the technology that powers them. Possible candidates for hosting are Google’s Compute Engine, AWS, and Rackspace. Part of this project is learning about what is out there, so I’ll need to do more investigation to determine which one of these I’d like to use and possible alternatives I don’t know about yet.

Web Server
==========
The web server portion of this project will give me an opportunity to brush up on my GoLang. The requirements for this server are simple: render a static webpage for a given request. A single page is probably fine, but I might want to have more than one for testing variety. The server should respond to both GET and POST requests the same way (again, variety is there to make monitoring and testing more interesting). The web server will be basic- it will process HTTP requests by responding with HTML pages.

API Server
==========
The API server will expose the same simple data set as the Web server will, but independent of HTML. It will return raw data to the client in the form of JSON or perhaps ProtoBuff for the client to consume.

A more advanced implementation of this system would be for the web server to farm out the logic of responding to requests to the API server. For example, in a sales application the a client using a browser could make a request for a page of items and their prices. The web server would make a call to the application server for the relevant information, then render it to the browser. Another client could query the application server directly via an API and retrieve pricing information directly, not embedded in HTML.

Both the API and Web servers will be written in GoLang.

Hammers
=======
The purpose of the hammers will be to hammer both the API and Web servers with requests. They should possess tunable parameters, eg have a frequency + amount of requests that can be changed to increase or decrease the traffic being sent to the servers. They should also be simple, just mechanisms to send requests and receive responses. Hammers should be written in Java.

Monitoring
==========

White Box
---------
To be able to monitor the system with full knowledge and access to its internal workings, I’ll want to set up some kind of white-box monitoring. Metrics of interest will be QPS, latency, errors, responses, and probably others that I’m not sure of at the moment. I’ll need to investigate possible monitoring solutions, as well as think about what it would take to write a few of my own, possibly implementing something like varz/streamz or some kind of counter + reporting metric classes in the serving code. I’m not sure whats out there that is commercially available and affordable for playing around with, so I’ll need to do some research.

Black Box
---------
Similar in implementation to the hammers, the black box monitoring will be performed by probers. The Probers will send a request every n time-units, and record the response. If too many probes have been unsuccessful in a predetermined time window, an alert should be sent.
The probers should be written in Java, and possibly a subclass of the Hammers.

Alerting
========
Again, this will require some research to determine what is out there as far as alerting solutions. I can also write my own to send notifications when certain error/latency thresholds are hit.

Logging
=======
More research required here. I know I’ll want sysdig installed on my servers, but I’m not sure what log aggregators are out there or what the best configurations are. I’ll have to look into it.

With all this in mind, the next steps are to create the simple servers in GoLang.
