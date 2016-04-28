---
layout: post
title: "Setup instantly local SMTP or HTTP server for debugging. Python to the rescue."
abstract: "Ever needed quickly setup HTTP or SMTP server for quick debugging of your application? Python is saving you, it has that prepared."
tags: python, smtp, http, testing
---

### HTTP server

The following command will serve contents of the folder where it is executed.

`python -m SimpleHTTPServer`

### SMTP Server

To start SMTP server on default port 25, you need sudo privileges, because 25 is reserved port in `<0, 1024>` range. Running following command will start SMTP process under `nobody` user.

`sudo python -m smtpd -c DebuggingServer localhost:25`

If you cannot get sudo privilege, you need to change listening port to something >1024. Still, process ownership is changed
to `nobody`, so if you run this on non-unix system, you need to avoid it.
This command will run SMTP server as current user on port 1025.

`sudo python -m smtpd -c DebuggingServer -n localhost:1025`
