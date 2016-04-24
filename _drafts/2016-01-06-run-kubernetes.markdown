---
layout: post
title:  "Single-node Kubernetes in Docker"
date:   2016-01-06 08:00:00
categories: containers
permalink: single-node-kubernetes-docker/
---

TODO:

- Use HEAD Hyperkube container image (build from source)
- Explain Docker and Kubernetes
- Explain difference between userspace and iptables kube-proxy

So as your 2016 resolution you promised to get deeper into all the DevOps shizzle going on? Turns out, Kubernetes and Docker are all the rage at the moment. One seemingly straightforward way to get in to that goodness is to run a single-node Kubernetes cluster in Docker.

First, some basics. Explanation of Docker & Kubernetes.

Let's use `lxc-checkconfig` to verify your kernel configuration:

{% highlight bash %}
$ dnf install lxc
$ lxc-checkconfig
{% endhighlight %}

If any of the configurations are missing, you need to enable them in the kernel. This should be easy if you have the kernel compiled with that feature enabled. On Grub2, just add `GRUB_CMDLINE_LINUX="cgroup_enable=memory"` in `/etc/default/grub` and run `update-grub2`. 

You can also check your kernel configuration:

{% highlight bash %}
$ cat /boot/config-`uname -r` | grep MEMCG
CONFIG_MEMCG=y
CONFIG_MEMCG_SWAP=y
CONFIG_MEMCG_SWAP_ENABLED=y
CONFIG_MEMCG_KMEM=y
{% endhighlight %}

Additionally, the Docker source code repository includes a shell script[^1] for checking your configuration.

[^1]:<https://github.com/docker/docker/blob/master/contrib/check-config.sh>

Now let's move on to Kubernetes-on-Docker[^2].

[^2]:<https://github.com/kubernetes/kubernetes/blob/master/docs/getting-started-guides/docker.md>

We'll build Docker images of the latest Kubernetes components. So:

{% highlight bash %}
$ git clone https://github.com/kubernetes/kubernetes.git
$ cd kubernetes
$ make release
{% endhighlight %}

This will build the Kubernetes components. You don't need to worry about build dependencies, because the build is containerized and will pull a Go environment.



Run a containerized etcd, which will act as a key/value storage for Kubernetes:

{% highlight bash %}
$ docker run --net=host -d gcr.io/google_containers/etcd:2.0.12 /usr/local/bin/etcd --addr=127.0.0.1:4001 --bind-addr=0.0.0.0:4001 --data-dir=/var/etcd/data
{% endhighlight %}

Run the primary Kubernetes container which will oversee running of the other services:

{% highlight bash %}
$ docker --version
Docker version 1.9.1-fc23, build 110aed2-dirty

$ docker run \
    --volume=/:/rootfs:ro \
    --volume=/sys:/sys:ro \
    --volume=/dev:/dev \
    --volume=/var/lib/docker/:/var/lib/docker:rw \
    --volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
    --volume=/var/run:/var/run:rw \
    --net=host \
    --pid=host \
    --privileged=true \
    -d \
    gcr.io/google_containers/hyperkube:v1.1.1 \
    /hyperkube kubelet \
        --containerized \
        --hostname-override="127.0.0.1" \
        --address="0.0.0.0" \
        --api-servers=http://localhost:8080 \
        --config=/etc/kubernetes/manifests \
        --allow-privileged=true --v=10
{% endhighlight %}

Also, run the service proxy:

{% highlight bash %}
docker run -d --net=host --privileged gcr.io/google_containers/hyperkube:v1.0.1 /hyperkube proxy --master=http://127.0.0.1:8080 --v=2
{% endhighlight %}

Download and configure kubectl:

Create a test nginx service:

That's it. In the following tutorial, we'll experiment on running a multi-node Kubernetes cluster locally with Docker. Expect heavy configuration ahead!
