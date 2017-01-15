---
layout: single
title: OpenStack Heat, Lessons Learned
published: false
---

As a follow-on and a learning exercise from the last post here, I decided to
use OpenStack Heat to fully orchestrate a deploy of MesosSphere's DC/OS. That
was about 3 weeks ago. I got it done, but wow, did I learn a lot the hard way.
The reason I started quest was to free up some hardware I had been using for
my Mesos install so I could install a newer version of OpenStack. My goal was
to fully automate the install and since MesosSphere has done a ton of work on
automation, I decided to use DC/OS.

Those of you not familiar with OpenStack Heat can find lots of documentation
online for it. A great starting point is
[Rackspace's Miguel Grinberg's Intro series](https://developer.rackspace.com/blog/openstack-orchestration-in-depth-part-1-introduction-to-heat/).

### Here is my list of notes and Lessons:

1. **YAML is fickle animal** - It is VERY easy to goof up the number of spaces you
   have and completely screw up your template, which leads to #2.

2. **Heat could use some better error messages and debugging capabilities** - When
   you completely screw up your template (see #1) and get error messages like
   the following from Heat it can be frustrating. It would be really nice it
   Heat would return some line numbers of something else to help you track this
   down.

   ```
   ERROR: Failed to validate: Property error: resources[0].properties: Unknown Property name
   ```

3. **Once it deploys errors can still crop up** - See
   [Steve Hardy's](http://hardysteven.blogspot.com/2015/04/debugging-tripleo-heat-templates.html)
   excelent blog post on how to debug nested templates. Steve works on the very
   complicated templates used to deploy OpenStack on OpenStack.

4. **Start small** - (or from an example) and work up, debugging is easier that way.
   Because of the lack of good messages as shown in #2 and #3 above, start with
   small parts of your template and make sure they work. Make one or a couple
   changes, so that when they fail you can remember what you changed.

5. **Indecision is killer** - I change my mind way too much on variable names and
   what to call things. It's a killer because there are so many names in a set
   of Yaml templates and I always miss one. I don't have a recommendation on
   how to solve this, but beware.

6. **Use Heat Software deployments takes some creative debugging** - If you use
   Heat's SoftwareDeployment model figuring out what happened can be much easier
   with a simple script. When one of these deployment resources run, it bundles
   up the output into a json file and put's it in /var/lib/heat-config/deployed
   on the instance it was working on.  The way it's formatted is difficult to
   read. A shortened example is shown here:

   ```
   {
     "deploy_stdout": "One Giant, long string.\n Concatenating together all of the lines from standard out.\n",
     "deploy_stderr": "Another Giant, super long string.\n Concatenating together all of the lines from standard error.\n",
     "deploy_status_code": 0
  }
   ```

   What helped a lot was I started putting this script in all of my deployments.

   ```
   resources:
     packageConfig:
       type: OS::Heat::CloudConfig
       properties:
         cloud_config:
           merge_how: 'dict(recurse_array,no_replace)+list(append)'
           write_files:
             - path: "/usr/local/bin/heat_debug_print"
               owner: "root"
               permissions: "0755"
               content: |
                 #!/bin/python
                 import json
                 from pprint import pprint
                 import argparse

                 parser = argparse.ArgumentParser(description='read large json and dump one field.')
                 parser.add_argument('file', type=argparse.FileType('r') )
                 parser.add_argument('field')
                 args = parser.parse_args()
                 with open(args.file.name) as data_file:
                     data = json.load(data_file)
                 print unicode(data[args.field])
   ```

   At which point I can run it and get the following.

   ```
  $ heat_debug_print /var/lib/heat-config/deployed/48b9b477-b4a1-4a01-a90c-0ed0fbdd5647.notify.json deploy_stderr
   Another Giant, super long string.
    Concatenating together all of the lines from standard error.

   ```

   Which is much easier to read. Remember this might be hundreds of lines long.
   Change the deploy_stderr to deploy_stdout to get standard out. As another
   note: You can get the same output json from 'heat deployment-show' but you
   still have the same concatenation and wrapping issues.

7. **Use 'depends_on' a lot** - With nested stacks you can pass references to objects using names or references. Heat can get confused and non-deterministic, which is not good. I learned my lesson, use 'depends_on' everywhere it's needed (always). If you run into the following, they might be fixed with proper 'depends_on':
  * Intermittent stack failures.
  * Passing keypairs and security groups by name reference to nested stacks.
  * Stack delete problems.

8. **SSH keypairs are annoying** - The deployment node for DC/OS needs to ssh to all nodes in the cluster. Heat runs all commands as root by default, which is what DC/OS wants too. Which means I needed the private key for root on the deployment node, but I also wanted to be somewhat secure and be able to login to be default account.

   This lead to the requirement that I have two keys on the default account, one for root@deployment_node and one for myself to get in.  To prevent putting my private key in the template (never do that), I generated a keypair for root@deployment_node, within the template, passed it to the instances and then added my public key afterwards.

  Three things are a pain here 1) Adding the private key to the root account took some magic with 'sed' and I still didn't get it to handle variable key lengths. 2) I had to manually stick the second key on the default user account. 3) The fact that most cloud instances do NOT allow one to directly ssh to root, is good and annoying at the same time.

  I'm sure there are better ways to do this, but my head was spinning by the time I understood what was needed and how to get it done.

9. **Heat is constantly changing** - Watch out for the Heat Domain Specific Language changes on each release of OpenStack. I was trying to make this all work on an OpenStack Mitaka and I read the document and found a great way to do something, BUT NO, it only works on Newton or better.  Here's the example:

   A Heat ResourceGroup is a way to create a number of servers, all the same. You can get a list back of the servers automatically and the Heat SoftwareDeploymentGroup is designed to deploy some software on a group of servers. BUT in Mitaka the list returned from ResourceGroup is not in the same format as the one needed by SoftwareDeploymentGroup.

   In the SoftwareDeploymentGroup one needs 'A map of names and server IDs to apply configuration to' and the output from ResourceGroup (In Mitaka) is simply a 'A list of resource IDs for the resources in the group.'

   In Newton someone added the attribute 'refs_map' which is exactly whats needed for the SoftwareDeploymentGroup, 'A map of resource names to IDs for the resources in the group.'

10. **'parameter_defaults' section is handy, but be careful** - This section of the environment file is great, but what makes it great also makes it dangerous. This section allows you to put all of the things you want to define into one place and eliminates the need to pass every parameter all the way down through the stack.  BUT what it does is basically pass everything to every stack (think global variables), so start giving your variables with very unique names or you will have clashes when using this feature.

Hopefully some part of this list can keep someone else from going crazy trying to figure this stuff out. I'm sure people will have comments, critiques and/or questions on how I did things. Send them along! Lastly if you want to take a look at the outcome of my struggle you can checkout the ever changing final product [here](https://github.com/ebalduf/Heat-mesos).
