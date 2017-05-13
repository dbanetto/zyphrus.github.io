---
layout: post
title:  "On demand Postgresql with systemd"
date:   2017-05-13 13:40:11 +12:00
categories: systemd ondemand linux
---

Having a local postgresql service on is fine when I'm working on a rails app
but I only want it on when I need it.
With systemd you can have some more control on when a service is started.
Today we are going to use a `.socket` file make postgresql start on demand.

What you need to do this is a Linux box that uses systemd and installed & setup postgresql.
Next you will need to create `postgresql.socket` (contents below),
put the file in `/etc/systemd/system` and give it `0644` (`-rw-r--r--`) permissions.

```systemd
; postgresql.socket
[Unit]
Description=Lazy initialisation of Postgresql

[Socket]
ListenStream=/run/postgresql/.s.PGSQL.5432

[Install]
WantedBy=sockets.target
```

To make this work will need to make sure you have disabled
postgresql from starting normally.
`systemctl disable postgresql.service` will stop postgresql from starting up by itself.

Then enable the use of our `postgresql.socket` use `systemctl enable postgresql.socket`.

Note the difference which type of `postgresql` service is being used,
before we disabled the `.service` which will start postgresql normally at boot
where the `.socket` just enabled will make systemd wait till there is activity on
that socket before starting the service.

This difference allows just to lazily load services on need instead of front loading
all of them at boot time.
The `.socket` works by camping on the resource described with `ListenStream`, could either
be a path to socket or a port number. Systemd will listen there until it gets any traffic
and starts up the service where it is supposed to go to. Systemd will also attempt to pass
over the data it received to the intended service.

There is a draw back with lazily loading services. Some services do not
support systemd's socket handover. This is the case for postgresql and will
fail to connect properly on the first attempt.
This really comes and bites when you are starting up your first rails app for
the day and `rails server` just stalls forever.
To get around this I generally run `psql` before starting any database related work.

This method can be extended to any other service that uses ports or
socket files (even remotely for ports), such as `openssh` ships with a `sshd.socket` (\*on Arch Linux) to allow
lazily starting ssh daemon.

> Note: this was tested on Arch Linux so the socket path for postgresql could differ for your
distribution.
