---
layout: single
title: Cinder Docker Driver with Mesos/Marathon
published: true
---

As a follow-on to two posts by my co-worker [John Griffith](https://github.com/j-griffith) around the Cinder Docker Driver, in this post I will take you through how to use the same plugin with Mesos/Marathon. John recently put together a Docker volume driver for OpenStack Cinder posted two great blog articles about how to use the driver with [Docker](http://j-griffith.github.io/cinder-providing-block-storage-for-more-than-just-nova/) and with [Docker swarm](http://j-griffith.github.io/cinder-providing-block-storage-for-more-than-just-nova-part-2/). In this blog post I'll cover how to setup and use the Cinder Docker Driver for Mesos/Marathon on Ubuntu and RHEL variants and be back with some additional details about CoreOS in a later post.

First a question: Should it be called the Cinder Docker Driver, or the Docker Cinder Driver?

### Prerequisites

We'll start by reviewing the installation steps. You will want to preform these steps on each and every Mesos agent in your environment. This could be a big task depending on your environment, so you might want to script (or automate) in some way. Since we will be using iSCSI for all volume communication, make sure the iSCSI initiator is installed.

For Ubuntu:

    sudo apt-get install open-iscsi

For RHEL variants:

    sudo yum install iscsi-initiator-utils

At this point, I would recommend using systemd to manage the Docker volume driver daemon and using John's great install script to get it done in one pass. You will need run this as root or use sudo.

```
curl -sSL https://raw.githubusercontent.com/j-griffith/cinder-docker-driver/master/install.sh \
| sudo sh
```

Next we'll need to setup a config file on every node and this is where the tricky parts come in since we'll need this file a slight bit different for every node. But first you may want to start by setting up a new project in OpenStack for Mesos. All examples here are with OpenStack Mitaka.

First create a mesos project

```
    # openstack project create mesos
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| enabled     | True                             |
| id          | c637cfebdc844837a6eb7011c4732255 |
| name        | mesos                            |
+-------------+----------------------------------+
```

Next create a mesos user under the project

    openstack user create --project mesos --password-prompt mesos

You may also need/want to manipulate the quotas for the project

    openstack quota set --volumes 20 --gigabytes 1000 mesos

Next let's look at that config file (/var/lib/cinder/dockerdriver/config.json). Mine looks like the following (from one node). You will need to customize the InitiatorIP (and I recommend the HostUUID) for each node. For the UUID I chose to use the last octet of the IP address after a delimited 'aaa' so I can keep track of which node has the volume mounted. Also notice I have pulled the TenantID from the openstack project create command.

```
    {
        "Endpoint": "http://172.27.31.11:5000/v2.0",
        "Username": "mesos",
        "Password": "mesospw",
        "TenantID": "c637cfebdc844837a6eb7011c4732255",
        "DefaultVolSz": 1,
        "HostUUID": "219b0670-a214-4281-8424-5bb3beaaa107",
        "InitiatorIP": "172.27.30.107"
}

```
If you want to automate the creation of the custom parts of the file, you can copy up the common partial file and use something like the following (subsituting VLAN1000 for your network interface) run on each node to fill in the custom parts.

```
    IPADDR=$(ip addr show | egrep 'VLAN1000' | grep -Eo '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' | head -1);IPLASTOCT=$(echo $IPADDR | awk -F'.' '{print $4}'); echo \"HostUUID\"\: \"219b0670-a214-4281-8424-5bb3beaaa$IPLASTOCT\", >> /var/lib/cinder/dockerdriver/config.json ;echo \"InitiatorIP\"\: \"$IPADDR\"\} >> /var/lib/cinder/dockerdriver/config.json
```

Once you have your file set, reload and restart systemd

    sudo systemctl daemon-reload
    systemctl start cinder-docker-driver

### Utilizing the plugin

Now lets build a simple application in Marathon requesting a single 'default' sized volume from Cinder. For the simple application, I'm going to use the python webserver at the root directory in our docker container, so we can explore what happend when the application was created.  Here is the json file in Marathon API to create the application. Notice the parameters passed to the Docker container to specify the driver (cinder) and the name of the volume (cinderDisk). The 'rm' parameter tells Docker to clean up the container when it terminates (and yes this works in Mesos 1.0).

```
{
  "id": "awebserveratroot",
  "cmd": "touch /cinderDisk/testfile; cd /;python3 -m http.server 8080",
  "cpus": 0.5,
  "mem": 64,
  "disk": 0,
  "instances": 1,
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "module/python-3-base",
      "network": "BRIDGE",
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 0,
          "protocol": "tcp",
          "labels": {}
        }
      ],
      "parameters": [
        {
          "key": "volume-driver",
          "value": "cinder"
        },
        {
          "key": "volume",
          "value": "cinderDisk:/cinderDisk"
        },
        {
          "key": "rm",
          "value": "true"
        }
      ],
      "forcePullImage": true
    }
  }
}
```

Go to the Marathon GUI
![MarathonGUI.png]({{site.baseurl}}/images/MarathonGUI.png)
(you can do this via the API too if you want) and click on the 'Create Application' button, then click on the 'JSON mode' in the upper right and paste the json replacing what already there. Launch the application. Once the application is running you should see the following in the Marathon GUI
![MarathonRunningApp.png]({{site.baseurl}}/images/MarathonRunningApp.png)
You can drill into the application and Marathon will provide information where (which node) it ran the application on, as shown below. In the below case it is running on container1.pm.solidfire.net at port 31548.

![MarathonAppLocation.png]({{site.baseurl}}/images/MarathonAppLocation.png)

If you click on the second line with the location, it will take you to the running application, our webserver, as shown here.

![WebserverListing.png]({{site.baseurl}}/images/WebserverListing.png)

You can see that our volume has been mounted at /cinderDisk and I'll leave it as a exercise to confirm our testfile is there.  Now, lets have a look at Cinder

```
# cinder list
+--------------------------------------+--------+------------+------+-------------+----------+--------------------------------------+
|                  ID                  | Status |    Name    | Size | Volume Type | Bootable |             Attached to              |
+--------------------------------------+--------+------------+------+-------------+----------+--------------------------------------+
| 99e5cd70-f277-47c1-976e-14e7a0524612 | in-use | cinderDisk |  1   |     lvm     |  false   | 219b0670-a214-4281-8424-5bb3beaaa106 |
+--------------------------------------+--------+------------+------+-------------+----------+--------------------------------------+
```

the volume has been created and is attached to a node (in my case the one with the last octet of it's IP = 106).

### Moving things around

One of the bigs functions Marathon provides is the ability to monitor applications and if the crash or have issues, restart them elsewhere in the Mesos cluster. The Cinder Docker driver fully supports this funcationality so login the node running the application and use Docker ps and Docker kill to terminate it. Let Marathon restart it somewhere and then find where and check that it moved the volume properly. I'll simply use Cinder to show it mounted at a different location.

```
[container1 ~]$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                     NAMES
197d2e31721e        module/python-3-base   "/bin/sh -c 'touch /c"   21 minutes ago      Up 21 minutes       0.0.0.0:31548->8080/tcp   mesos-b15f11b9-fdbf-48e4-a148-bd1073249887-S0.2396850f-5b03-49c4-af99-a04bf7780a5d
[container1 ~]$ docker kill 197d2e31721e
197d2e31721e

# cinder list
+--------------------------------------+--------+------------+------+-------------+----------+--------------------------------------+
|                  ID                  | Status |    Name    | Size | Volume Type | Bootable |             Attached to              |
+--------------------------------------+--------+------------+------+-------------+----------+--------------------------------------+
| 99e5cd70-f277-47c1-976e-14e7a0524612 | in-use | cinderDisk |  1   |     lvm     |  false   | 219b0670-a214-4281-8424-5bb3beaaa107 |
+--------------------------------------+--------+------------+------+-------------+----------+--------------------------------------+
```

As an exercise you can kill this application, modify the json and remove the 'touch testfile' from the commmand line and then restart the application to prove that the volume contents remain unchanged.

### Other stuff

One of the things you will notice is that the volume is of the size defined in the 'DefaultVolSz' line of the config file. Docker designed their interfaces with the idea of explicit volume creation, what we have done is implicitly created the volume on container start.  What Docker really wants us to do is to create this volume before we create the application. To do that, we'll kill our application (Through the Marathon GUI) and then we'll login to one of our Mesos nodes and remove the volume, explicitly create a new volume with the same name, specifying the size and type of the volume on the create command.  Then start the application again, pointing it to the volume by name and it will just pickup that volume, mount, and use it.

```
$ docker volume ls
DRIVER              VOLUME NAME
cinder              cinderDisk
$ docker volume rm cinderDisk
cinderDisk
$ docker volume create -d cinder --name=cinderDisk -o size=20 -o type=solidfire
cinderDisk

# cinder list
+--------------------------------------+-----------+------------+------+-------------+----------+-------------+
|                  ID                  |   Status  |    Name    | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+------------+------+-------------+----------+-------------+
| 0eb2c2f8-87f5-477c-a6cd-e1d22dc84132 | available | cinderDisk |  20  |  solidfire  |  false   |             |
+--------------------------------------+-----------+------------+------+-------------+----------+-------------+
```


