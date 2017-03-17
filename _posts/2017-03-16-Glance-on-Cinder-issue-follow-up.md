---
layout: single
title: Using cinder for Glance images - fail - fix.
published: true
---

Today is my last day at the current employer, and of course I'm spending the
minutes figuring out why the post from 2 weeks ago doesn't work. The big
clue is that the customer is using RedHat OSP10 (Newton) so it should work, but
NO. Here are the symptoms:

```
$ os image create --public --file trusty-server-cloudimg-amd64-disk1.img --disk-format qcow2 ubuntu_on_cinder
503 Service Unavailable
Insufficient permissions on image storage media: Permission to write image storage media denied.
    (HTTP 503)
```

In the glance log:

```
2017-03-16 21:27:28.395 653781 DEBUG os_brick.utils [req-0605a5f3-fce7-4592-a0de-fc4f3a3884d2 ca8b37482e0042b5a72b7d6a855f0035 7ce1fcab9a1b4b4d8dc573d5e6aa9263 - default default] ==> get_connector_properties: call {'execute': None, 'my_ip': 'overcloud-controller-0.pm.solidfire.net', 'enforce_multipath': False, 'host': None, 'root_helper': 'sudo glance-rootwrap /etc/glance/rootwrap.conf', 'multipath': False} trace_logging_wrapper /usr/lib/python2.7/site-packages/os_brick/utils.py:141
2017-03-16 21:27:28.400 653781 DEBUG os_brick.utils [req-0605a5f3-fce7-4592-a0de-fc4f3a3884d2 ca8b37482e0042b5a72b7d6a855f0035 7ce1fcab9a1b4b4d8dc573d5e6aa9263 - default default] <== get_connector_properties: exception (3ms) error(13, 'Permission denied') trace_logging_wrapper /usr/lib/python2.7/site-packages/os_brick/utils.py:151
2017-03-16 21:27:28.401 653781 ERROR glance_store._drivers.cinder [req-0605a5f3-fce7-4592-a0de-fc4f3a3884d2 ca8b37482e0042b5a72b7d6a855f0035 7ce1fcab9a1b4b4d8dc573d5e6aa9263 - default default] Failed to write to volume f55ca0c7-a170-4c6a-8f8a-3029e0d14b76.
2017-03-16 21:27:30.124 653781 DEBUG oslo_messaging._drivers.amqpdriver [req-0605a5f3-fce7-4592-a0de-fc4f3a3884d2 ca8b37482e0042b5a72b7d6a855f0035 7ce1fcab9a1b4b4d8dc573d5e6aa9263 - default default] CAST unique_id: d8549dede1904fe68633e8cd81da330c NOTIFY exchange 'glance' topic 'notifications.error' _send /usr/lib/python2.7/site-packages/oslo_messaging/_drivers/amqpdriver.py:432
2017-03-16 21:27:30.132 653781 ERROR glance.api.v2.image_data [req-0605a5f3-fce7-4592-a0de-fc4f3a3884d2 ca8b37482e0042b5a72b7d6a855f0035 7ce1fcab9a1b4b4d8dc573d5e6aa9263 - default default] Failed to upload image data due to HTTP error
```

Note the 'Permission denied' message. When I traced this through the code, it
was in the rootwrap facility or actually the prisep facility as the daemon was
attempting to start and was attempting to create a linux socket in a randomly
generated /tmp directory. After some googling, a co-worker of mine discovered
this bug https://bugzilla.redhat.com/show_bug.cgi?id=1395240
which is exactly the problem.

The solution is to implement the selinux module
from the .te file attached to that bug. The procedure I used is as follows.

```
checkmodule -M -m -o glance-api.mod glance-api.te
semodule_package -o glance-api.pp -m glance-api.mod
semodule -i glance-api.pp
```

Hope that helps. I'll potentially post some good stuff from the next job,
depending if what I'm doing is cool.
