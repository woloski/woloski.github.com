---
layout: post
title: Why node.js on Windows? (for Microsoft centric devs)
tags : [node, windows]
---
{% include JB/setup %}

Yesterday, on an internal discussion list, someone said:

> as a long-time .NET developer I don't see the allure of node. We already have lots of framework support for scalability/async as well as very good tooling and library support. My sense is that node.js has a lot of catch-up to do

Here is my answer:

I don't think node.js is only about the non-blocking capabilities. That's what made it popular and novel in the beginning. However, there are many other reasons why you would want to use node.js:

* **Community/open source**: The node.js open source community is vibrant. Nuget, as of today, has 5751 package (from <http://stats.nuget.org/>). Node has 9,450 (from <http://search.npmjs.org/>). It's amazing the things you can reuse and the quality is really good. Just to pick an example: we developed MarkdownR, which is a collaborative markdown editor, by mashing up things like ShareJS, snowdown, ace and azure storage. ShareJS implements OT ([Operational Transformation](http://en.wikipedia.org/wiki/Operational_transformation)) which is the same principle used in Google Waves to merge operations from multiple concurrent clients. It builds on top of browserchannel and socket.io (which is kind of the SignalR equivalent in node). With this I want to illustrate the kind of things (high level building blocks) that you can rely on.

* **One language to rule them all**: It's JavaScript all over. This could be good or bad for some people, but arguably there are many devs out there who love JavaScript. And if you mix it with mongo, you have the full stack. If you want to take it further, I recommend you to look at this: <http://meteor.com/>

* **Multi-device/multi-platform**: It runs on Windows and Linux and the benchmarks are pretty good. This makes it a great choice for develop software that runs on cloud and on-premise on any device (ARM/x86/x64) and platform (even on a $35 usd computer, [Raspberry PI](http://blog.tomg.co/post/21322413373/how-to-install-node-js-on-your-raspberry-pi)). For instance, I am currently developing a logging/auditing server on node and mongo. I can run it on Azure and provide the "Enterprise" version running on the customer premises (or even customer cloud) and it's the same code. I could even setup a server with the same code in a different cloud (AWS, Heroku, nodejitsu, etc.) within minutes. More flexibility.

* **developer friendly**: For development, you can use Mac, Linux or Windows with a simple text editor like Sublime Text 2 or go with WebMatrix to get IntelliSense. All free.

* **NoSQL**: If you are interested in NoSql, you have tons of options and drivers for node.js.

* **Mix and match**: And to make it even more interesting, you can run node.js as part of an ASP.NET solution with IIS node (mix and match). So it's not an all or nothing decision.

Then [Glenn Block](http://codebetter.com/glennblock/) jumped into the conversation and added:

* **Lightweight**: It is also extremely lightweight. In node you start with a single exe on Windows, or command on Mac. It’s hard to really grock that until you actually use it. 

* **Designed for real time**: Aside from that another very interesting aspect is that node was designed specifically with real time web apps in mind i.e. WebSockets etc. There are some very compelling modules for building realtime applications which is one of the sweet spots for the community around node.


I also recommend reading this post by Rick who also comes from a Microsoft perspective and compares the processing model of IIS against the event-loop from node.js: <http://rickgaribay.net/archive/2012/01/28/node-is-not-single-threaded.aspx>

