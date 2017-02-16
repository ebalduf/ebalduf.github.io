---
layout: single
title: Configure OpenStack Glance (images) to use Cinder for storage.
published: true
---

In OpenStack the project called Glance provides the catalog of images to use to
create VMs. At first everyone assumes Glance 'is' the storage for those images,
but in fact Glance is just the catalog of where those images are stored and
uses one of the other mechanisms available in OpenStack to store the images.
Most setups configure Glance to store the images in Swift or other object
storage, but there are options, including file, NFS, and Ceph. If you are using
a single OpenStack controller there isn't a requirement for shared storage, but
when you have a HA configuration for your OpenStack controllers, there is a
requirement for the storage to be shared. I had one customer configure GFS2
shared amongst the OpenStack controllers with a SAN volume backing it.

Until Mitaka (where it was listed experimental) the ability to store images in
Cinder (block storage) was broken or non-existent. In Newton the code has
stabilized more, so lets look at what it takes to configure Glance to use
Cinder. I'm going to do this all in a Newton devstack, but it should work in
any Newton and beyond (i.e. Ocata)

There is a big decision one needs to make when they configure glance to use
Cinder. The decision involves who owns the cinder volumes and therefore who pays
for them (You are charging for access to storage in your cloud now aren't you?).
So here are you two choices:
1. All images are owned by a project/tenant, which admin sets up, just for
images.
2. Each project/tenant owns the cinder storage for images.
Remember Glance maintains it's own ACL for each image, so no matter which one
you pick above Glance can swizzle access as needed.

## Option 1: All images owned by one project.

To configure glance to use cinder in this scenario start by creating a user and
project to own the cinder volumes.

```
$ alias os=openstack
$ os project create glance_cinder
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| enabled     | True                             |
| id          | 25af9b57d8f3431dad1c4bc86c8a2ae9 |
| name        | glance_cinder                    |
+-------------+----------------------------------+
$ os user create --project glance_cinder --password glance_cinder_pw glance_cinder
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| email      | None                             |
| enabled    | True                             |
| id         | 43940e60b6c0486997ebfd8121403ed3 |
| name       | glance_cinder                    |
| project_id | 25af9b57d8f3431dad1c4bc86c8a2ae9 |
| username   | glance_cinder                    |
+------------+----------------------------------+
```

Then set the following options in /etc/glance/glance-api.conf
```
[glance_store]
stores = file, http, swift, cinder
default_store = cinder
cinder_state_transition_timeout = 300
cinder_store_auth_address = http://172.27.162.247:5000/v2.0
cinder_store_user_name = glance_cinder
cinder_store_password = glance_cinder_pw
cinder_store_project_name = glance_cinder
rootwrap_config = /etc/glance/rootwrap.conf
cinder_catalog_info = volumev2::publicURL
```
Lets talk about a few of these:
- Notice you can have multiple backends. I'm adding 'cinder' in this example.
- I define cinder as the default.
- The username, password and project name (lines 6-8) are what you created
above.
- The rootwrap stuff on line 9 might be there already, or you might need to
create it, see below.
- Line 10 causes Glance to ask Keystone how to reach cinder, if you want to
explicitly configure how that connection should happen, leave off that line and
add the following 3 lines.
```
cinder_os_region_name
cinder_endpoint_template
cinder_store_auth_address
```

Next, also in glance-api.conf, you need to define a strategy for which backend
to use when. As you can see below, there is a preference order and then there
is a location_strategy variable which you can either go with the order things
are stored in glance "location_order" or the priority you have given in the store_type_preference variable, which is indicated by setting "store_type" as
I have done below.
```
[store_type_location_strategy]
store_type_preference = cinder, swift, http, file
location_strategy = store_type
```
At this point, restart the glance-api service then you can store some images.
Here's the example output.
```
$ . openrc admin admin
$ os image create --public --file ../trusty-server-cloudimg-amd64-disk1.img --disk-format qcow2 ubuntu_on_cinder
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 16f58bf9d54a995d86007e669df38383                     |
| container_format | bare                                                 |
| created_at       | 2017-02-15T16:45:02Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/a0735468-022b-4997-84ab-b2c716885e87/file |
| id               | a0735468-022b-4997-84ab-b2c716885e87                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | ubuntu_on_cinder                                     |
| owner            | 7a1d59f7e6d046a0b76b7d8206715a52                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 261227008                                            |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2017-02-15T16:45:23Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
$ os image list
+--------------------------------------+---------------------------------+--------+
| ID                                   | Name                            | Status |
+--------------------------------------+---------------------------------+--------+
| a0735468-022b-4997-84ab-b2c716885e87 | ubuntu_on_cinder                | active |
| c2910de4-3201-4e68-a151-effa8c9fcca8 | cirros-0.3.2-x86_64-disk        | active |
+--------------------------------------+---------------------------------+--------+
$ os image show a0735468-022b-4997-84ab-b2c716885e87
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 16f58bf9d54a995d86007e669df38383                     |
| container_format | bare                                                 |
| created_at       | 2017-02-15T16:45:02Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/a0735468-022b-4997-84ab-b2c716885e87/file |
| id               | a0735468-022b-4997-84ab-b2c716885e87                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | ubuntu_on_cinder                                     |
| owner            | 7a1d59f7e6d046a0b76b7d8206715a52                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 261227008                                            |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2017-02-15T16:45:23Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
$ cinder list --all-tenants
+--------------------------------------+----------------------------------+-----------+--------------------------------------------+------+-------------+----------+
| ID                                   | Tenant ID                        | Status    | Name                                       | Size | Volume Type | Bootable |
+--------------------------------------+----------------------------------+-----------+--------------------------------------------+------+-------------+----------+
| dc085ff0-f32d-4279-8d1e-684482c7fab5 | 25af9b57d8f3431dad1c4bc86c8a2ae9 | available | image-a0735468-022b-4997-84ab-b2c716885e87 | 1    | lvmdriver-1 | false    |
+--------------------------------------+----------------------------------+-----------+--------------------------------------------+------+-------------+----------+
```
Notice in the last output that cinder shows a single volume with a display name
which includes the image ID. You will notice that the Volume Type is using my
current employer's storage, and that is because glance does not currently send a
volume type to cinder, so cinder will pick the configured default type.  If you
con't have it one set or you don't like the choice, go into cinder.conf and set "default_volume_type=solidfire".  Now if you want to can boot a server like
normal from the image.
```
$ os server create --image a0735468-022b-4997-84ab-b2c716885e87 --flavor 2 --security-group default --key-name balduf-wlaptop --nic net-id=dae0e28e-e2d5-4c81-a122-0fcf00a2c3ca test_ubuntu_from_cinder
```

## Option 2: Each project owns their images.

It is really simple to switch this to the other method whereby each project owns
the cinder volumes for their images. Simply comment out these 3 lines from the glance-api.conf
```
#cinder_store_user_name = glance_cinder
#cinder_store_password = glance_cinder_pw
#cinder_store_project_name = glance_cinder
```
Restart the glance-api service. Lets look at how it works now.

```
$ . openrc demo demo
$ os image create  --file ../trusty-server-cloudimg-amd64-disk1.img --disk-format qcow2 ubuntu_on_cinder-demo
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 16f58bf9d54a995d86007e669df38383                     |
| container_format | bare                                                 |
| created_at       | 2017-02-16T03:09:54Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/15dc8033-335a-4311-8f56-4769b8a17061/file |
| id               | 15dc8033-335a-4311-8f56-4769b8a17061                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | ubuntu_on_cinder-demo                                |
| owner            | faf590becbd24f72a14226ef1eb7afa3                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 261227008                                            |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2017-02-16T03:10:11Z                                 |
| virtual_size     | None                                                 |
| visibility       | private                                              |
+------------------+------------------------------------------------------+
[ebalduf@devstack-newton devstack]$ cinder list
+--------------------------------------+-----------+--------------------------------------------+------+-------------+----------+
| ID                                   | Status    | Name                                       | Size | Volume Type | Bootable |
+--------------------------------------+-----------+--------------------------------------------+------+-------------+----------+
| 834f39e7-6dfd-44c0-af6b-705e9e0de5b7 | available | image-15dc8033-335a-4311-8f56-4769b8a17061 | 1    | solidfire   | false    |
+--------------------------------------+-----------+--------------------------------------------+------+-------------+----------+
$ . openrc another another
$ cinder list
+----+--------+------+------+-------------+----------+-------------+
| ID | Status | Name | Size | Volume Type | Bootable | Attached to |
+----+--------+------+------+-------------+----------+-------------+
+----+--------+------+------+-------------+----------+-------------+
$ os image create  --file ../trusty-server-cloudimg-amd64-disk1.img --disk-format qcow2 ubuntu_on_cinder-another
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 16f58bf9d54a995d86007e669df38383                     |
| container_format | bare                                                 |
| created_at       | 2017-02-16T03:10:55Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/edc43d0a-c237-4867-b3c5-2c142dfc2448/file |
| id               | edc43d0a-c237-4867-b3c5-2c142dfc2448                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | ubuntu_on_cinder-another                             |
| owner            | b0527431ef144c038844288436903456                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 261227008                                            |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2017-02-16T03:11:11Z                                 |
| virtual_size     | None                                                 |
| visibility       | private                                              |
+------------------+------------------------------------------------------+
$ cinder list
+--------------------------------------+-----------+--------------------------------------------+------+-------------+----------+
| ID                                   | Status    | Name                                       | Size | Volume Type | Bootable |
+--------------------------------------+-----------+--------------------------------------------+------+-------------+----------+
| ecdb37c3-6164-4133-acf8-4fb5c4d86620 | available | image-edc43d0a-c237-4867-b3c5-2c142dfc2448 | 1    | solidfire   | false    |
+--------------------------------------+-----------+--------------------------------------------+------+-------------+----------+
$ . openrc admin admin
$ cinder list --all-tenants
+--------------------------------------+----------------------------------+-----------+--------------------------------------------+------+-------------+----------+
| ID                                   | Tenant ID                        | Status    | Name                                       | Size | Volume Type | Bootable |
+--------------------------------------+----------------------------------+-----------+--------------------------------------------+------+-------------+----------+
| 834f39e7-6dfd-44c0-af6b-705e9e0de5b7 | faf590becbd24f72a14226ef1eb7afa3 | available | image-15dc8033-335a-4311-8f56-4769b8a17061 | 1    | solidfire   | false    |
| ecdb37c3-6164-4133-acf8-4fb5c4d86620 | b0527431ef144c038844288436903456 | available | image-edc43d0a-c237-4867-b3c5-2c142dfc2448 | 1    | solidfire   | false    |
+--------------------------------------+----------------------------------+-----------+--------------------------------------------+------+-------------+----------+
```

Notice where I source the openrc file to change users (and projects) and which
cinder volumes are created when I great my images. ** A big word of warning here
** the user could make a mess by deleting the volume using cinder.  

** Another note: ** This is left as an exercise to the user, notice I'm not
marking the images above 'public' as my current setup doesn't allow non admin's
to mark images public. If I wanted to make any of these public I have to mark
them as the *admin* user or create them as *admin* with the --public flag, in
which case *admin* owns the cinder volumes (and pays).

### A Note about rootwrap

One thing that a co-worker of mine ran into when setting this up at a customer
site is the lack of a /etc/glance/rootwrap.conf file. These rootwrap files are
used prevent the various projects from abusing the root priviledges they need
for certain tasks. Basically they are list of allowed tasks the project can
preform. In this case, glance needs to be able to mount volumes.  The simplest
way to accomplish this is to create a rootwrap.conf file which gives glance the priviledges of cinder. If you distribution doesn't have the file, create the
following file /etc/glance/rootwrap.conf with these contents.

```
[DEFAULT]
filters_path=/etc/cinder/rootwrap.d
exec_dirs=/sbin,/usr/sbin,/bin,/usr/bin,/usr/local/bin,/usr/local/sbin
use_syslog=False
syslog_log_facility=syslog
syslog_log_level=ERROR
```

### Caveat

One problem with all of this, is that Glance has a new V2 API and when they
ported the glance client to use that API they took away the ability to direct
images to your favorite backed. The API still has the ability to define the
backend in the request, but the client has removed the option. The answer, for
now, is to force the glance V1 API from the command line. Here is how:

```
$ glance --os-image-api 2 help image-create | grep "\-\-store"
# No option
$ glance --os-image-api 1 help image-create | grep "\-\-store"
usage: glance image-create [--id <IMAGE_ID>] [--name <NAME>] [--store <STORE>]
  --store <STORE>       Store to upload image to.
```

The glance V1 support is marked for removal in a future release.  Perhaps in the
future V2 will also have a backend store option. I have filed a bug against this
removal which you can track [here](https://bugs.launchpad.net/python-glanceclient/+bug/1665208).

#### Lastly ####
Here is my complete glance-api.conf for example 1 (without comments).

```
$ grep '^[^#]' /etc/glance/glance-api.conf
[DEFAULT]
logging_exception_prefix = %(color)s%(asctime)s.%(msecs)03d TRACE %(name)s %(instance)s
logging_debug_format_suffix = from (pid=%(process)d) %(funcName)s %(pathname)s:%(lineno)d
logging_default_format_string = %(asctime)s.%(msecs)03d %(color)s%(levelname)s %(name)s [-%(color)s] %(instance)s%(color)s%(message)s
logging_context_format_string = %(asctime)s.%(msecs)03d %(color)s%(levelname)s %(name)s [%(request_id)s %(user)s %(tenant)s%(color)s] %(instance)s%(color)s%(message)s
graceful_shutdown_timeout = 5
workers = 2
registry_host = 172.27.162.247
transport_url = rabbit://stackrabbit:secret@172.27.162.247:5672/
image_cache_dir = /opt/stack/data/glance/cache/
use_syslog = False
bind_host = 0.0.0.0
debug = True
location_strategy = store_type
[cors]
allowed_origin = http://172.27.162.247
[cors.subdomain]
[database]
connection = mysql+pymysql://root:secret@127.0.0.1/glance?charset=utf8
[glance_store]
stores = file, http, swift, cinder
default_swift_reference = ref1
swift_store_config_file = /etc/glance/glance-swift-store.conf
swift_store_create_container_on_put = True
default_store = swift
filesystem_store_datadir = /opt/stack/data/glance/images/
default_store = cinder
cinder_catalog_info = volumev2::publicURL
cinder_state_transition_timeout = 300
cinder_store_auth_address = http://172.27.162.247:5000/v2.0
cinder_store_user_name = glance_cinder
cinder_store_password = glance_cinder_pw
cinder_store_project_name = glance_cinder
rootwrap_config = /etc/glance/rootwrap.conf
[image_format]
[keystone_authtoken]
memcached_servers = 172.27.162.247:11211
signing_dir = /var/cache/glance/api
cafile = /opt/stack/data/ca-bundle.pem
auth_uri = http://172.27.162.247/identity
project_domain_name = Default
project_name = service
user_domain_name = Default
password = secret
username = glance
auth_url = http://172.27.162.247/identity_v2_admin
auth_type = password
[matchmaker_redis]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_notifications]
driver = messaging
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
flavor = keystone+cachemanagement
[profiler]
[store_type_location_strategy]
store_type_preference = cinder
[task]
[taskflow_executor]
```
