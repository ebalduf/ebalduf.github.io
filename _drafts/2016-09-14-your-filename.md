---
layout: default
title: Hacking a blog together
published: true
---

Test post with a code block and image.

	echo $(ip addr show | egrep 'VLAN1000|eno16780032' | grep -Eo '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' | head -1)

![mountainbike.gif]({{site.baseurl}}/_posts/mountainbike.gif)
