+++
author = "Saharsh Singh"
title = "Let's Talk Containers"
slug= "lets-talk-containers"
date = "2018-04-18"
description = "While looking and feeling like virtual machines, containers work best when addressing packaging and deployment of an individual process."
tags = [
    "containerization",
    "devops",
    "software",
    "tech",
    "work",
]
categories = [
    "articles",
]
aliases = [
    "/2018/04/18/lets-talk-containers/",
]
+++

This is not another tutorial or introductory article about Docker containers. Instead, I want to use this article to try and help newcomers reach an “Aha!” moment sooner than later. And that moment is the realization that while looking and feeling like virtual machines, containers work best when addressing packaging and deployment of an individual process.

<!--more-->

Containers are everywhere. I use them extensively in both my day job and my random side projects. Almost every serious software shop on the planet is either using containers today or strongly considering them. When trying to get my coworkers and other developer friends to jump on the bandwagon, I often joke that if container based deployments are in your future, then you are living in the past. Obviously that’s a tongue-in-cheek comment and containers are not a silver bullet that solve every deployment or packaging concern. However, they are one of the biggest paradigm shifts I have seen in my career. And I am a guy who hates terms like “paradigm shifts”.

My experience with containers so far is both extensive and limited. It’s extensive in that I have used containers to address quite a few different concerns. Obviously I have containerized web services that can be easily deployed in production environments using container orchestration tools like Kubernetes or ECS – probably their most popular use case. I have heavily leveraged containers and tools like docker-compose to enable massive productivity gains when setting up fully integrated local development environments. Containers have been a godsend for me when establishing appropriate integration testing gateways for my applications’ CI/CD pipleines. Containers have also shown a lot of promise in replacing periodic background tasks running on provisioned servers (costly) with autoscaled tasks that only turn on resources when actually needed. And, honestly, I can keep going. On the other hand, my experience with containers is limited in that I have only ever used **Docker**. I have not personally used **rkt** or various other lower level alternative container technologies like **LXD**. Given the ubiquitousness of Docker and having read some comparisons between the alternatives, I don’t think I’m missing out. Still, I wanted to mention that before we dig in. Okay, let’s dig in.

As I said earlier, this is not a tutorial on containers. If you are looking to learn from the ground up, there are some great resources out there. From [this slightly old blog article](https://medium.freecodecamp.org/a-beginner-friendly-introduction-to-containers-vms-and-docker-79a9e3e119b) that does an excellent job of defining containers to [official Docker documentation](https://docs.docker.com/engine/docker-overview/), there’s plenty of existing literature out there to get you started. In this article, I want to address something specific I see people new to containers struggle with. Containers are awesome, because, using specific **Linux kernel features** – **namespaces, cgroups, and union file systems** – they provide processes complete **user space isolation** on the same host. This can make them look and feel like virtual machines, but there is a key difference. Virtual machines provide virtualization at the hardware level allowing you to run multiple full blown operating system instances on the same hardware. A [hypervisor](https://en.wikipedia.org/wiki/Hypervisor) is needed to do this. In fact, you may remember having to specifically turn on virtualization features in your computer’s BIOS before being able to use virtual machines. Containers, on the other hand, are primarily a Linux technology. Until recently – Windows has started adding similar [container friendly features](https://blog.docker.com/2016/09/build-your-first-docker-windows-server-container/) as the Linux kernel – they were uniquely a Linux technology. Docker for Mac and Windows get around this by installing a light weight Linux VM where all the Docker containers actually execute. What does this mean? This means that Docker containers are just plain old processes that share the same kernel but enjoy user space isolation to virtualize operating system instances. So, while you have the ability to choose from multiple operating systems for your container, you should have noticed that only Linux operating systems are supported (or only the Windows ones in Docker’s Windows runtime environment).

Here we come to a moment similar to when, in the first Matrix movie, Neo asks Morpheus, “What are you trying to tell me? That I can dodge bullets?”. To this Morpheus replies, “No, Neo. I’m trying to tell you that when you’re ready, you won’t have to.” I see many container newbies trying to approach creation of new container images the way they would approach configuring a virtual machine before a snapshot or, worse, a full blown bare metal. To beat this article’s dead horse, **containers are not virtual machines**. Yes, you can replace a VM completely via a single container image, and there might even be a valid use case for that. However, approaching containerization in that view only will severely limit your imagination and creativity in what you can actually do with the technology. The ideal use case of containers is to package a single application and all its, and only its, required dependencies (e.g. runtime libraries) into a portable, disposable, and, most importantly, self-sufficient container image. In fact, the minimalist Linux distributions like [Alpine](https://alpinelinux.org/about/) have exploded in popularity within the Docker community for this very reason. Obviously, self-sufficiency here applies to runtime libraries and other operating system specific features, not configuration. Most real world server side applications rely on environment specific configuration, and the image should not include it, but rather expose tightly controlled interfaces to accept it. For examples of that, check out [environment variables and volume mounts](https://docs.docker.com/engine/reference/commandline/run/) in Docker.

Now, virtual machines are not going anywhere. As I mentioned above, you need a Linux VM to even run Docker containers on a Mac or a Windows. Most PaaS providers that support containers will run your containers on one or more VMs, and will allow you to configure the horsepower backing them to ensure your containers have the required memory and processing power. Hell, you will need to make sure you allocate enough memory and CPUs for the Linux VM running on your Mac or Windows machine to support your local container load (try running multiple database container images with only 1GB allocated to your Docker daemon). However, virtual machines do take a massive backseat. They are no longer required to provide every possible dependency needed by every possible process that might ever run on them. Their file systems no longer have to cater to the deployment footprint of every process. They just need to provide a shared kernel and enough horsepower, and container instances will take care of the rest themselves (with lots of help from their engineers, of course).
