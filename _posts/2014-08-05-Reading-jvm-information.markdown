---
layout: post
title: "Reading JVM information from hperfdata files"
abstract: "When monitoring JVM processes, using jps and jstat tools might me little bit too heavy"
published: true
---
*{{page.abstract}}*

-----

Thanks to [Twitter commons](https://github.com/twitter/commons/) tools, it is possible to read JVM parameters with Python instead of Java. In my case, I was building a tool that scans all JVM processes (and also those that are appearing during execution) and perodically reads heap characteristics to make overview of memory consumption for each of these JVMs. This is very close to what JConsole or VisualVM does on desktop, but this tool was intended to run on server, unattended, automatically started.

When process is detected (I am using `/proc` file system scanning to get new Java processes) my first version of this tool used to execute first `jps` to get the name (either main class or jar archive) of such process. Then it periodically executed `jstat` to get heap characteristics. This solution works quite well, only it gets stuck from time to time when there is too much load on the system.

*Note:* I was running `jstat` on perodic basis as there were some hurdles in getting output from batched execution.

But I was not satisfied with this solution and also, I was wondering how it is possible that `jps` knows main class even when 4k limit for `cmdline` file is reached in `/proc` file system, e.g. when extremelly long classpath is used. Luckily, my [question on StackOverflow](http://stackoverflow.com/questions/25079439/how-does-jps-tool-get-the-name-of-the-main-class-or-jar-it-is-executing) got its answer. `jps` knows it from *hsperfdata* files gathered by JVM for each Java process.

I started to investigate ways to parse *hsperfdata* files and as my monitoring app is written in Python, that was my area of investigation. Luckily, I found above mentione **Twitter common** library that does exactly what I need and does it in an eye blink. The following snippet is a Linux-dependent (`tempfile.gettempdir()` will most probably work, but I have no Win machine to test with) version of `jps`. Thanks Twitter.

<script src="https://gist.github.com/martinhynar/702acf820da2290b4f75.js"></script>

