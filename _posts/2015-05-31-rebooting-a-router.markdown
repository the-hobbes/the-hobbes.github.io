---
layout: post
title:  "Rebooting a home router with Python."
date:   2015-04-18 16:48:47
categories: python scripting router
---
 I was sitting with my laptop this weekend watching a BBC documentary about Henry the 8th (it really is amazing where you can end up on the end of a wikipedia click-binge) when the internet went out. This is an anonoying thing that happens from time to time with my router, an older Netgear WNDR3400.


<h3> Context: </h3>

Every so often, the connection between it and my macbook is lost, and logs are filled with errors about portal disassociations and whatnot (I'd 
provide a few error messages here as examples, but unfortunately my logfile turned over at about 10:30 this morning due to a size limit being met).

The connection gets lost and although you can briefly connect again, the wifi card and the router will stop talking after a few seconds.
It isn't a huge deal as a reboot of the router will cause everything to work fine (until it happens again), but it is annoying. Instead of
rebooting the router manually, I've taken to just doing what amounts to a no-op update of the settings by going to the router's web interface and clicking the "apply" button on the basic menu. This causes the router to reboot and the packets to flow once more.

I had some extra time on my hands and figured I'd automate the router rebooting process. Better yet, I'd make the router reboot on a 
regular schedule so I didn't have to worry about it. I decided to whip up a python script to do just that, and run it with cron.

<h3>But Phelan, why don't you just...</h3>

Q: Root-cause the issue with the router?

A: I've looked at it in the past, and I think it is probably just a function of the age of the router and the wireless chipset in my mac not playing well together. Also, troubleshooting tech issues at home sucks when you do it all day at work.

Q: Just reboot the thing manually?

A: And get up from the couch? Surely you must be joking. 

Q: Get a new router?

A: This one works fine most of the time! It would be an act of betrayal to give up on my loyal packet-shepherd so easily. 

Q: Install DD-WRT?

A: While the Netgear WNDR3400 is a supported router, I really didn't want to run the risk of bricking my device. Plus, writing a script sounded
like a way more fun project than reflashing firmware (that's just me though).

Q: Ok, if you insist on a script why not use Selenium to make it for you? 

A: I considered it, and [Selenium][selenium] is cool, but I really just wanted to write some python, you know?

<h3>Digging around:</h3>

The Netgear WNDR3400 doesn't have any kind of API that you can send RPCs to, so I decided I would automate what I had been doing in the first place: do a reboot by clicking the "Apply" button with the default settings in the web UI. The first thing I did was take a look at the source HTML for the router administration page.

The "Apply" button that I was clicking lived in a form element with a particular target. The form and button elements both looked like this:

{% highlight html %}
<form name="formname" method="POST" action="ether.cgi?id=1139543405" target="_parent"> 
<input type="SUBMIT" name="apply" value=Apply onClick="return checkData()">
{% endhighlight %}

So now I knew that "Apply" was sending a POST request to a cgi script at "ether.cgi?id=1139543405". This gave me a target at which to send
information, but next I needed to understand what that data I had to send was going to be.

In Chrome, I opened up the developer tools and headed to the network tab to examine the POST request that the form was making. I clicked
"Apply" and examined the headers as they were sent to the router. The most relevent headers were the "Request Headers", which showed me information
about the request itself, and the "Form Data", which showed me what was being sent to the router by the form (these included things
like the system name, hardware address, IP Addres, etc...). The most important field to me was the "apply:Apply" section, presumably
indicating this was a request to apply the data received to the system and thus necessitating a reboot.

Now I had all the information I needed to start scripting (or so I thought).

<h3>The script:</h3>
Since my router is password protected, the first order of business was to implement authentication for the script. This was done by using 
urllib2.HTTPPasswordMgr and urllib2.HTTPBasicAuthHandler to generate an "opener" object that could be used to authenticate the connection
with my username and password. I used curl to get the http headers that would provide the authentication ["realm" information][realm], like so:

{% highlight bash %}
$ curl -I http://192.168.1.1/start.htm
HTTP/1.0 401 Unauthorized
WWW-Authenticate: Basic realm="NETGEAR WNDR3400"
Content-type: text/htm
{% endhighlight %}

Once I had functions to perform both the authentication and the connection work, I experimented with seeing whether or not I could request the
relevent html page from the router (the page with the form and "Apply" button on it). It all worked and I got the proper html printout, so I moved on to the function that would perform the POST operation to reboot the router.

I was able to authenticate to the POST url I had pulled from the headers in Chrome's developer tools, but I kept receiving 404 responses when I
tried to connect. Confused, I eventually broke out [Postman][postman], a Chrome extension that lets you send HTTP requests and examine their
responses. When I tried the POST request directly through Postman, I noticed that the url I had copied from the form's
target had changed. The 10-digit id code was different, and I confirmed this by making several "apply" actions and looking at the newly 
generated forms. The ID changed each time. 

This meant a little extra work, as I now needed to grab the ID from the form first in order to know where the router was expecting the POST's
to come in on. After I had the right url the 404's went away, but I was presented with a new error: 

{% highlight bash %}
httplib.BadStatusLine: ''
{% endhighlight %}

I solved this by catching the httplib.BadStatusLine error and ignoring whatever strange response the Netgear was giving me. After I did that, I was
able to successfully POST to the router and remotely reboot it. There was much rejoicing!

<h3>The cron:</h3>
To automate the script, I added an entry in my crontab. This can be done by running the "crontab -e" command and editing the crontab file, using the cron syntax to specify the target action you'd like to perform and its frequency. I wanted my script to run every Sunday at 6pm, so I set
it up to look like this (I also added a guide in the comments at the top of the file): 

{% highlight bash %}
# +--------- Minute (0-59)                    | Output Dumper: >/dev/null 2>&1
# | +------- Hour (0-23)                      | Multiple Values Use Commas: 3,12,47
# | | +----- Day Of Month (1-31)              | Do every X intervals: */X  -> Example: */15 * * * *  Is every 15 minutes
# | | | +--- Month (1 -12)                    | Aliases: @reboot -> Run once at startup; @hourly -> 0 * * * *;
# | | | | +- Day Of Week (0-6) (Sunday = 0)   | @daily -> 0 0 * * *; @weekly -> 0 0 * * 0; @monthly ->0 0 1 * *;
# | | | | |                                   | @yearly -> 0 0 1 1 *;

# reboot my router every sunday at 6pm
0 18 * * 0 python ~/bin/routerReboot.py
{% endhighlight %}

Once you've saved, if the cron is installed correctly you can view it by using "crontab -l". Additionally, cron will send an email to your local user account to tell you that it has completed/failed. To send this message to an email account you actually check, you can add a ".forward" file to your home directory, and in this file place an email address to where you would like to recieve cron reports.

<h3>Conclusion:</h3>
You can find [my script here][script]. A lot of headers I'm using probably aren't needed, but I just did a copy-paste of the header information in from the developer tools. The same likely goes for the form data, but I'm not sure if the router is smart enough to not overwrite stored default data with null/0 values if you don't give it anything. Erring on the side of caution, I decided to just feed it all the defaults. 


[selenium]: http://www.seleniumhq.org/
[realm]:    http://tools.ietf.org/html/rfc2617#page-3
[postman]:  https://chrome.google.com/webstore/detail/postman-rest-client/fdmmgilgnpjigdojojpjoooidkmcomcm?hl=en
[script]:   https://github.com/the-hobbes/router_reboot/blob/master/routerReboot.py
