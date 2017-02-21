---
layout: single
title: OpenStack Heat tricks - Part 1 of unknown.
published: true
---

A quick post so I remember this, and maybe someone else doesn't have to fight to
figure this out.

Heat can create very long hostnames and linux has a limit of 64 characters. Heat
will often times exceed this limit if you don't configure it not to. Prior to
Newton there was no easy method, you had to hack the code for the Heat engine,
specifically edit (exact location depends on your distro)

```
/usr/lib/python2.7/dist-packages/heat/engine/resources/openstack/nova/server.py
```

look for physical_resource_name_limit at about line 563 and change the value
from 53 to something which allows for your hostname with your domain name to
come in under 64.

In Newton, this was changed to a configurable, so you can just change the value
max_server_name_length in /etc/heat/heat.conf

**Note:** If you leave you cloud domain name set at novalocal then 53 should be
a good answer.  On the other hand if you change it to something like
xx.solidfire.net this is when you start to run into issues.
