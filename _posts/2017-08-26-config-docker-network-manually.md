---
layout: post
title:  "How To Config Your Docker Networking Manually"
date:   2017-08-26
tag:
- Edward 
- Docker
- Network
- IP
---
I am looking for the article to introduce the container network since I met many problem when deploy
my kubernetes cluster. After I detected many blogs and books I found a way to config your docker network
manually on linux os, I'd like share this here, hopes that can help who are looking for.

Before we share the way that to detect or config the docker network, firstly, let me lead you to take a
look at the linux network namespace and its command **ip**. ip is a powerful tool of network configuration on
linux, it can not only do what the traditional tool e.g. ifconfig, route, iptales do, but it can config the
network namepace.

#### operation the network namespace via **ip netns**
* create a namespace:
{% highlight css %}
edward@fresky:~# sudo ip netns add test1
{% endhighlight %}

* list the namespaces
{% highlight css %}
edward@freesky:~# sudo ip netns list
test1
{% endhighlight %}

* delete a namespaces
{% highlight css %}
edward@freesky:~# sudo ip netns delete test1
{% endhighlight %}

* execute a command inner namespace
{% highlight css %}
edward@freesky:~# sudo ip netns exec test1 ip link
{% endhighlight %}

You may notice that we can do almost everything related linux namespace you want with ip netns, now one question
come up, can we use it as docker network config tool? the answer is yes.

However, there are several proparing things should be finished before we can use it as the docker network tool.
that because the the ip netns will search the namespace under the path /var/run/netns that is not the docker work
folder. we mush map the docker network namespace to the ip netns work directory. 

* find the process id of the docker instance running
{% highlight css %}
edward@freesky:~# sudo docker inspect --format '{{ .State.Pid }}' mycontainer 
3243
{% endhighlight %}

* link to ip netns directory
{% highlight css %}
edward@freesky:~# sudo ln -s /proc/3243/ns/net /var/run/netns/mycontianer
{% endhighlight %}

if the directory /var/run/netns isn't exsit, create it first: sudo mkdir -p /var/run/netns 
{: .notice}

* test 
{% highlight css %}
edward@freesky:~# sudo ip netns list
<container>
{% endhighlight %}

Now, all that is ready, you can use the ip netns to config the docker network as you expect.
according to my experience, I'd like start a bash window for my work as follow:
{% highlight css %}
edward@freesky:~# sudo ip netns exec mycontainer bash
{% endhighlight %}
quit the namespace bash via inputing 'exit' after all work is fine.

please mark label it is from freesky-edward's blog if you plan to forward this blog, thanks a lot.
{: .notice}
 
