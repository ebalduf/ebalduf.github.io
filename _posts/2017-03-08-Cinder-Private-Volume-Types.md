---
layout: single
title: OpenStack Cinder (Block storage) Private Volume Types.
published: true
---

Another quick post because I have been asked about this a couple times in the 
last 2 weeks.  This is documented in the OpenStack documentation, but not in a
way that people seem to find, read and understand. Hopefully, this explanation
will be clearer.

The request that I keep getting is to allow some projects to see a volume-type
while others only see the basic type catalog. This capability has been in Cinder
since Kilo, but it has never made it into Horizon and like a mentioned above is
for some reason overlooked and misunderstood.

Everything I do here will be with a Newton install of OpenStack (RHEL OSP 10).
I am also going to attempt to show as much of this using the openstack command
as possible, but this will show it's clearly still a work in progress, as I am
forced to go back and use the cinder command a couple times.  **Note:** I have
aliased 'openstack' to 'os'.

Lets start with a quick view of what I have already.

```
[RHEL-OSP10 ~]$ . overcloudrc-admin
[RHEL-OSP10 ~]$ os volume type list
+--------------------------------------+-----------+
| ID                                   | Name      |
+--------------------------------------+-----------+
| ff38af61-d32f-4eec-bdf2-1adfa17ad9fb | silver    |
| 4b8649f5-6570-4ca9-ad74-f3698dd91449 | gold      |
+--------------------------------------+-----------+
[RHEL-OSP10 ~]$ os volume qos list
+--------------------------------------+---------------+----------+--------------+--------------------------------------------------------------+
| ID                                   | Name          | Consumer | Associations | Specs                                                        |
+--------------------------------------+---------------+----------+--------------+--------------------------------------------------------------+
| 540481de-6914-4fda-8638-9b2c0b649310 | platinum-qos  | back-end | platinum     | qos:burstIOPS='2450', qos:maxIOPS='2250', qos:minIOPS='2000' |
| 1f2c3630-b868-4f72-8b7a-084b8693d096 | silver-qos    | back-end | silver       | qos:burstIOPS='400', qos:maxIOPS='350', qos:minIOPS='300'    |
| d9123dbd-9580-403c-8300-5f920caa2eaa | gold-qos      | back-end | gold         | qos:burstIOPS='1500', qos:maxIOPS='1250', qos:minIOPS='1000' |
+--------------------------------------+---------------+----------+--------------+--------------------------------------------------------------+
[RHEL-OSP10 ~]$ cinder qos-get-association d9123dbd-9580-403c-8300-5f920caa2eaa
+------------------+------+--------------------------------------+
| Association_Type | Name | ID                                   |
+------------------+------+--------------------------------------+
| volume_type      | gold | 4b8649f5-6570-4ca9-ad74-f3698dd91449 |
+------------------+------+--------------------------------------+
```
As you can see I have 2 volume types (silver and gold) and 3 QoS specifications
(silver, gold and platinum) and well make a private volume-type for platinum for
one of my 2 projects.

```
[RHEL-OSP10 ~]$ os volume type create --private --property volume_backend_name=solidfire platinum
+---------------------------------+--------------------------------------+
| Field                           | Value                                |
+---------------------------------+--------------------------------------+
| description                     | None                                 |
| id                              | d1f98207-86e5-4fba-b6a9-07b19c41ab10 |
| is_public                       | False                                |
| name                            | platinum                             |
| os-volume-type-access:is_public | False                                |
| properties                      | volume_backend_name='solidfire'      |
+---------------------------------+--------------------------------------+
[RHEL-OSP10 ~]$ os volume qos associate 540481de-6914-4fda-8638-9b2c0b649310 d1f98207-86e5-4fba-b6a9-07b19c41ab10
```
The above creates the volume-type, marking it private and then associates the
QoS specification with it. Next is where the magic happens, we need to
create an access list for this private volume-type to allow one of my 2 projects
to see it.
```
[RHEL-OSP10 ~]$ os project list
+----------------------------------+----------------+
| ID                               | Name           |
+----------------------------------+----------------+
| 7a41518fa7c84c8fbc9c496dc0ead2f2 | anotherProject |
| db1adcb48b594ddca082ff1d53a4f2d8 | demo           |
+----------------------------------+----------------+
[RHEL-OSP10 ~]$ cinder type-access-add --volume-type d1f98207-86e5-4fba-b6a9-07b19c41ab10 --project-id 7a41518fa7c84c8fbc9c496dc0ead2f2
[RHEL-OSP10 ~]$ cinder type-access-list --volume-type d1f98207-86e5-4fba-b6a9-07b19c41ab10
+--------------------------------------+----------------------------------+
| Volume_type_ID                       | Project_ID                       |
+--------------------------------------+----------------------------------+
| d1f98207-86e5-4fba-b6a9-07b19c41ab10 | 7a41518fa7c84c8fbc9c496dc0ead2f2 |
+--------------------------------------+----------------------------------+
```
As you can see, for this you need to use 'cinder' commands. Next let's look at
this from the perspective of my two projects.
```
[RHEL-OSP10 ~]$ . overcloudrc-demo
[RHEL-OSP10 ~]$ os volume type list
+--------------------------------------+-----------+
| ID                                   | Name      |
+--------------------------------------+-----------+
| 4b8649f5-6570-4ca9-ad74-f3698dd91449 | gold      |
| ff38af61-d32f-4eec-bdf2-1adfa17ad9fb | silver    |
+--------------------------------------+-----------+
[RHEL-OSP10 ~]$ os volume create --type d1f98207-86e5-4fba-b6a9-07b19c41ab10 --size 1 attempt-at-platinum
Volume type d1f98207-86e5-4fba-b6a9-07b19c41ab10 could not be found. (HTTP 404) (Request-ID: req-478e5fbf-ff7b-41e1-9655-5d7452cd5504)
[RHEL-OSP10 ~]$ . overcloudrc-another
[RHEL-OSP10 ~]$ os volume type list
+--------------------------------------+-----------+
| ID                                   | Name      |
+--------------------------------------+-----------+
| d1f98207-86e5-4fba-b6a9-07b19c41ab10 | platinum  |
| 4b8649f5-6570-4ca9-ad74-f3698dd91449 | gold      |
| ff38af61-d32f-4eec-bdf2-1adfa17ad9fb | silver    |
+--------------------------------------+-----------+
[RHEL-OSP10 ~]$ os volume create --type d1f98207-86e5-4fba-b6a9-07b19c41ab10 --size 1 a-platinum-vol
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
...
| id                  | c9115836-d4f7-4eaf-8902-cb5c607ece66 |
...
| type                | platinum                             |
+---------------------+--------------------------------------+
```
As you can see, the demo user cannot access the platinum type, even if they
happen to get the volume-type_ID. The 'anotherUser' user can create volumes with
the volume-type 'platinum'  

Now lets have a look at Horizon. Horizon works fine for the users as seen here
from the perspective of the 'anotherUser' user. The users see the private
volume-type like normal.

![PrivVolUser.png]({{site.baseurl}}/images/PrivVolUser.png)

On the other hand in the Newton version of Horizon, the admin panel has some
problems once you associate a QoS specification with the volume-type. You will
get this error ![AdminError.png]({{site.baseurl}}/images/AdminError.png) and the
Admin / System/ Volumes / Volume Types page will not show any Volume Types in
the first table as seen here:

![AdminPanelErr.png]({{site.baseurl}}/images/AdminPanelErr.png)

I have confirmed this in a Newton devstack too.  The good news is that in an
Ocata devstack, I can confirm it has been fixed as seen here:

![OcataAdminPanelworks.png]({{site.baseurl}}/images/OcataAdminPanelworks.png)

In summary, private volume-types work great and can be used for some interesting
scenarios, but since the feature is not one everyone uses often, there are some
rough edges to be aware of.

I'm off to my next adventure and may or may not be writing any more about
OpenStack. Hopefully these posts help someone.
