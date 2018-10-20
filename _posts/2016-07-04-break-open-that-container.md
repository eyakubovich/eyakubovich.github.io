---
layout: post
title: Break open that container!
date: '2016-07-04T12:34:00.000-07:00'
author: Eugene Yakubovich
tags:
- Linux
- Containers
modified_time: '2016-07-04T12:36:39.385-07:00'
---

If you happen to follow the world of IT infrastructure, DevOps, or just happen to be an IT professional, you, no doubt, have heard of the container revolution.
Propelled into popularity by Docker, containers (and the ecosystem around them) seem to be the answer to all things ops.
I, too, like the elegance with which containers solve many of the problems around shipping and running software.

Great, but let's pause for a second to see what problems containers actually solve.
I contend that they solve three distinct problems:
1. Ability to package software in its entirety and ship it from the publisher to the consumer.
Basically it's about distributing a tarball with all the bits and then running those bits without worrying what else is on the target machine.
It's solves the dependency hell problem that package mangers were never quite able to do.

2. Running the software in isolation from other processes/containers on the system. Your contained service can now have its own PID space and its very own TCP port 80.
If you happen to blindly schedule services onto machines (e.g. if using an orchestration system like Kubernetes), you can sleep soundly knowing that the services won't conflict.

3. Constraining the resources. When running a container, you can cap the CPU, memory, and I/O that the processes inside the container are allowed to consume.
As I've said before, these are three distinct problems and yet all the container systems try to couple them into a single solution.

This is unfortunate for two reasons:
1. Often times, I do not need all three when running a service or an application. In fact, most often I care just about (1) -- pulling an image and running it, without worrying about the dependencies that will get sprinkled onto my system.
Or I might want to dedicate a machine to my production database and thus do not need isolation or resource constraints.
Yet I would still like to use Docker hub to install the bits.
Docker, rkt, systemd-nspawn and others do provide knobs to turn off bits and pieces but it is more painful than it needs to be.
On the flip side, I may want to isolate and constrain an application that I installed via apt-get or yum (and no, unshare(1) and cgexec(1) do not quite cut it).

2. It does not allow independent innovation in these areas.
The coupling prevents me (without hoop jumping) from using Docker to pull and execute the app while using rkt for resource isolation.
This feels very un-UNIXy.

It is easy to see how we got to where we are.
Containers were initially viewed as lightweight VMs and so inherited the same semantics as their heavier cousins.
VMs, in turn, coupled the aforementioned areas due to their implementations.
I think it is safe to say that most people no longer view containers as lightweight VMs.
The notion of a "systems container" has been superseded by an "application container".
It would be prudent for organisations like [Open Container Initiative](https://www.opencontainers.org/) to consider defining interfaces that would allow innovation of individual components of the container runtimes.
