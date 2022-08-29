+++
author = "Saharsh Singh"
title = "Buildah, Podman, and Skopeo"
slug= "buildah_podman_skopeo"
date = "2019-01-18"
description = "In this article, I explore the exciting new world of rootless and daemon-less Linux container tools."
tags = [
    "containerization",
    "devops",
    "howto",
    "kubernetes",
    "software",
    "tech",
    "work",
]
categories = [
    "articles",
]
aliases = [
    "/2019/01/18/buildah_podman_skopeo/",
]
+++

Still doing all your Linux container management using an insecure, bloated daemon? Well, don’t feel bad. I was too until very recently. Now I’m finding myself slowly saying goodbye to my beloved Docker daemon and saying hello to Buildah, Podman, and Skopeo. In this article, I explore the exciting new world of rootless and daemon-less Linux container tools.

<!--more-->

## Some History

If you are anything like me, **Docker** and **containers** might as well be the same word. **Dockerfiles** and `build`, `pull`, `run`, `tag`, and `push` commands of the **Docker CLI** have become part of my daily life as a software developer for the last few years. However, days of the monolith daemon approach to container management are pretty much coming to an end. There have been two big major factors leading to this. First, **Docker** and [CoreOS](https://coreos.com/), along with few other organizations, started [Open Container Intiative (OCI)](https://www.opencontainers.org/) as an attempt to standardize **container runtime** and **image format** specifications in 2015. OCI efforts have since matured quite a bit. The OCI image format specification is now supported by most, if not all, mainstream image registries (e.g. Docker Hub, Amazon’s ECR, etc). The OCI runtime specification has a reference implementation in **runc**, and most existing container runtimes are either OCI compliant already or have OCI compliance in their roadmap.

Second, **Kubernetes** created **CRI** to start supporting vendors other than Docker for their container runtime implementation. The [story behind the creation of CRI](https://medium.com/cri-o/container-runtimes-clarity-342b62172dc3) is actually pretty amusing. Essentially, in the beginning, Docker was the **only supported runtime** in Kubernetes. Naturally, CoreOS wanted **rkt** support in Kubernetes, and therefore submitted a pull request that basically had a bunch of “if some-flag do rkt else do docker“. Kubernetes saw this approach breaking the [open-closed principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle) and decided to do the right thing by creating CRI as a high level abstraction to be implemented by vendors like Docker, CoreOS, and whoever else. Meanwhile, the landscape of the container ecosystem changed in one significant way. Docker had enjoyed its status as the de facto runtime and image format standard in Linux containers from 2013 till about 2017. Their image format and runtime were constants in an otherwise heated war in the container orchestration space. However, since 2017 or so, Kubernetes has decidedly won that war and become the de facto standard in container orchestration.

All these developments mean Docker is suddenly finding itself drifting away from the center of the Linux container universe. Instead of enjoying other projects orbiting its enormous gravity well, Docker is slowly finding itself becoming one of the many others orbiting the gravity well of Kubernetes. This excites some people who always saw the monolith daemon that required root access for everything as a problem. Which brings us to the heart of this article – the daemon-less and largely rootless suite of container management tools.

## New Generation of Container Management Tools

[Buildah](https://github.com/containers/buildah) builds, [Podman](https://github.com/containers/libpod) runs, and [Skopeo](https://github.com/containers/skopeo) transfers container images. These are open source tools maintained by the [containers](https://github.com/containers) organization on Github. There is some overlap in functionality between Buildah and Podman, but the separation of core responsibilities is clear. None of these tools require a running daemon, and for the most part don’t need root access either (you will see one exception to the rootless rule in Podman in this article). So, having set the stage, I’ll spend the rest of the article sharing how I quickly ramped up on the three tools using my recent dummy application, Task Master, from my [Gophin Off with Linked Lists](/2018/12/18/gophin-off-with-linked-lists-part-1/) series. You don’t need to go through that series for this article. Basically Task Master is a dumb little web app written in Go that I will containerize and play with using these tools. Grab the source code [here](https://github.com/saharsh-samples/gophinoff/tree/master/taskmaster).

Before we get started, one caveat with these tools is that they are very much Linux tools. This should be obvious given they are dealing with Linux containers. After all, Docker itself is specific to Linux. However, Docker does a good job of hiding behind a VM in MacOS and Windows to give those user bases a seamless experience. These tools, being fairly new on the block, don’t yet have those conveniences. You will have to grab a Linux VM if you want to play with them on your Windows or MacOS machines. OK let’s get started.

## Containerizing using Docker

I wrote the Task Master web app trying to learn Golang. So, I haven’t yet bothered containerizing it yet. So, before jumping in to the new tools, let’s containerize the app using good ol’ Docker. Dockerfile for a simple Go app is pretty straightforward.

{{< highlight dockerfile "linenos=table" >}}
FROM scratch

LABEL maintainer="Saharsh Singh"

COPY taskmaster /

EXPOSE 8080
ENTRYPOINT [ "/taskmaster" ]
{{< /highlight >}}

The Dockerfile assumes there is a standalone executable already present in the top level of the Dockerfile context. It copies it to the root directory of the container file system. It advertises `8080` as a port to expose, since that’s the port where Task Master listens for incoming HTTP requests. Finally, the copied over Task Master executable is set as the entry point. Alright, so now I need to build the executable for Task Master so that the Docker build can run. To keep things seamless, I’ll go ahead and write a build script so that I can just run one command to build both the Go executable as well as the Docker image.

{{< highlight bash "linenos=table" >}}
#!/bin/bash

# Define script constants
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

# Change directory to code location
set -e
pushd $SCRIPT_DIR
trap popd EXIT INT TERM

# Test code
go test ./...

# Build Go binary
time GOOS=linux CGO_ENABLED=0 go build

# Build container
docker build -t saharshsingh/taskmaster .
{{< /highlight >}}

I still haven’t figured out how to run the Go test and build commands safely while being in another directory. The compiler seems to have weird issues with fully qualified file paths. So that’s why you see the pushd, trap, and popd business here. Otherwise things are straightforward.

* Run unit tests.
* Build the standalone executable targeting Linux.
* Build the container image.

Alright time to run this bad boy.

```
[sahsingh@sahsingh gophinoff]$ ./taskmaster/build.sh
~/go/src/github.com/saharshsingh/gophinoff/taskmaster ~/go/src/github.com/saharshsingh/gophinoff
?       github.com/saharshsingh/gophinoff/taskmaster    [no test files]
?       github.com/saharshsingh/gophinoff/taskmaster/http       [no test files]
ok      github.com/saharshsingh/gophinoff/taskmaster/impl       (cached)
 
real    0m0.640s
user    0m0.655s
sys     0m0.216s
Sending build context to Docker daemon  6.901MB
Step 1/5 : FROM scratch
 --->
Step 2/5 : LABEL maintainer="Saharsh Singh"
 ---> Running in 565c237d6254
Removing intermediate container 565c237d6254
 ---> c21fc573607e
Step 3/5 : COPY taskmaster /
 ---> ee354893b8d4
Step 4/5 : EXPOSE 8080
 ---> Running in dce9a0919d54
Removing intermediate container dce9a0919d54
 ---> 586259b53a99
Step 5/5 : ENTRYPOINT [ "/taskmaster" ]
 ---> Running in 05f3b3249c44
Removing intermediate container 05f3b3249c44
 ---> 58e6754cfcb0
Successfully built 58e6754cfcb0
Successfully tagged saharshsingh/taskmaster:latest
~/go/src/github.com/saharshsingh/gophinoff
Saharshs-MBP:gophinoff ssingh$
Saharshs-MBP:gophinoff ssingh$
Saharshs-MBP:gophinoff ssingh$ docker images
REPOSITORY                                               TAG                 IMAGE ID            CREATED              SIZE
saharshsingh/taskmaster                                  latest              58e6754cfcb0        About a minute ago   6.87MB
```

Great! Do note the awesomely minimal container image size of just 6M. Go go! Alright, let’s do some light testing.

```
[sahsingh@sahsingh gophinoff]$ docker run -d -p 8080:8080 --name taskmaster saharshsingh/taskmaster
5f38d3f13314717a283f60485edcb15a225818e8391c3fff7e9856620c225d29
Saharshs-MBP:gophinoff ssingh$
Saharshs-MBP:gophinoff ssingh$
Saharshs-MBP:gophinoff ssingh$ curl -i localhost:8080/task && echo
HTTP/1.1 200 OK
Content-Type: application/json
Date: Fri, 18 Jan 2019 18:47:22 GMT
Content-Length: 4
 
null
[sahsingh@sahsingh gophinoff]$ curl -i localhost:8080/task -XPOST -H "Content-Type: application/json" -d '{"Name":"Some Task"}' && echo
HTTP/1.1 201 Created
Date: Fri, 18 Jan 2019 18:48:04 GMT
Content-Length: 0
 
 
[sahsingh@sahsingh gophinoff]$ curl -i localhost:8080/task && echo
HTTP/1.1 200 OK
Content-Type: application/json
Date: Fri, 18 Jan 2019 18:48:07 GMT
Content-Length: 50
 
{"Name":"Some Task","Description":"","Priority":0}
[sahsingh@sahsingh gophinoff]$ docker stop taskmaster && docker rm taskmaster && docker rmi saharshsingh/taskmaster
taskmaster
taskmaster
Untagged: saharshsingh/taskmaster:latest
Deleted: sha256:58e6754cfcb0124a7475b10986cac9600eb5eee0308adf7ff011be7eb0fe31e8
Deleted: sha256:586259b53a994ff7d9b6af31d6fcf9016ab9a8ab0ca54ba56652dcb26b8baeac
Deleted: sha256:ee354893b8d47c592551a80dccc8254d816452d9750b6415c1b444a9029aae27
Deleted: sha256:223399315075be73ac5e761f349e7d223859ee35ff4c0c08658af236bdbb1c1d
```

Ok, so that’s just plain old Docker. Knowing that things work, let’s convert this over to use our new tools.

## Installing the tools

Installation of these tools is very straightforward. Just grab your Linux package manager and install the `buildah`, `podman`, and `skopeo` packages. Being on Fedora, I use `dnf`.

```
[sahsingh@sahsingh gophinoff]$ sudo dnf -y install buildah podman skopeo
```

## Buildah Using Dockerfiles

So the cool thing about migrating to Buildah is that it supports Dockerfiles. So the easiest thing to do here is to use Buildah’s `bud` (short for `build-using-dockerfile`) command.

```
[sahsingh@sahsingh gophinoff]$ buildah bud -t saharshsingh/taskmaster taskmaster
STEP 1: FROM scratch
STEP 2: LABEL maintainer="Saharsh Singh"
STEP 3: COPY taskmaster /
STEP 4: EXPOSE 8080
STEP 5: ENTRYPOINT [ "/taskmaster" ]
STEP 6: COMMIT containers-storage:[vfs@/home/sahsingh/.local/share/containers/storage+/run/user/1000:overlay.mount_program=/usr/bin/fuse-overlayfs]localhost/saharshsingh/taskmaster:latest
Getting image source signatures
Copying blob sha256:258da81a6b2aba1003628a44cec752cf44067f74a299abe1c636acd25d8654c5
3.30 MiB / 3.30 MiB [======================================================] 0s
Copying config sha256:9d82e14463d8eec0c408a5245435f3fd74ca0055243a96c2cb76d49d49ff93ce
438 B / 438 B [============================================================] 0s
Writing manifest to image destination
Storing signatures
--> 9d82e14463d8eec0c408a5245435f3fd74ca0055243a96c2cb76d49d49ff93ce
[sahsingh@sahsingh gophinoff]$
[sahsingh@sahsingh gophinoff]$
[sahsingh@sahsingh gophinoff]$ buildah images
IMAGE NAME IMAGE TAG IMAGE ID SIZE
localhost/saharshsingh/taskmaster latest 9d82e14463d8 6.87 MB
```

Note how only one image layer, as opposed to one layer per step in Docker, is created. And just like that we have our image. Do note that I am using the `buildah images` command to see my image. I have, in fact, gone ahead and uninstalled Docker on my machine completely. This image is also saved underneath my home directory instead of `/var/lib`. What this means is this image is local to my user on this machine. If I run `buildah images` as any other user on this machine, I won’t see that image. Quick way to illustrate this is using `sudo` to build the image as `root`.

```
[sahsingh@sahsingh gophinoff]$ buildah rmi 9d8
9d82e14463d8eec0c408a5245435f3fd74ca0055243a96c2cb76d49d49ff93ce
[sahsingh@sahsingh gophinoff]$ sudo buildah bud -t saharshsingh/taskmaster taskmaster
[sudo] password for sahsingh:
STEP 1: FROM scratch
STEP 2: LABEL maintainer="Saharsh Singh"
STEP 3: COPY taskmaster /
STEP 4: EXPOSE 8080
STEP 5: ENTRYPOINT [ "/taskmaster" ]
STEP 6: COMMIT containers-storage:[overlay@/var/lib/containers/storage+/var/run/containers/storage:overlay.mountopt=nodev,overlay.override_kernel_check=true]localhost/saharshsingh/taskmaster:latest
Getting image source signatures
Copying blob sha256:258da81a6b2aba1003628a44cec752cf44067f74a299abe1c636acd25d8654c5
3.30 MiB / 3.30 MiB [======================================================] 0s
Copying config sha256:207306a078e79c60650e68acc10ac7ea2944e2d5e79a1bdddbed9c50937499c1
438 B / 438 B [============================================================] 0s
Writing manifest to image destination
Storing signatures
--> 207306a078e79c60650e68acc10ac7ea2944e2d5e79a1bdddbed9c50937499c1
[sahsingh@sahsingh gophinoff]$ buildah images
[sahsingh@sahsingh gophinoff]$ sudo buildah images
IMAGE NAME IMAGE TAG IMAGE ID SIZE
localhost/saharshsingh/taskmaster latest 207306a078e7 6.87 MB
```

Note how buildah images shows no images while sudo buildah images shows the image we just built.

## Buildah beyond Dockerfiles

Buildah is much more than just a third party tool to process Dockerfiles however. Buildah allows you to interactively build container images one step at a time. It does this by spawning an instance of the container from some base image. You can then use this container to execute all the necessary steps to get to your final image or some intermediate layer. Once you are done with a layer of the build, you can commit the container up to that point as an image tag to Buildah, and restart the process from that tag as the base image. Once you are completely done, commit the final tag and remove the working containers.

To see this in practice, let’s recreate the way Docker built our image using Buildah. First we spawn a working container using the `scratch` image.

```
[sahsingh@sahsingh gophinoff]$ buildah from scratch
working-container
[sahsingh@sahsingh gophinoff]$ buildah containers
CONTAINER ID BUILDER IMAGE ID IMAGE NAME CONTAINER NAME
41b2e348f98a * scratch working-container
[sahsingh@sahsingh gophinoff]$
```

Alright, so we have our container instance named working-container to work with. Next let’s create the maintainer label, commit the change, and spawn another intermediate container.

```
[sahsingh@sahsingh gophinoff]$ buildah config --label "maintainer=Saharsh Singh" working-container
[sahsingh@sahsingh gophinoff]$ buildah commit working-container tm-intermediate-1
Getting image source signatures
Copying blob sha256:4f4fb700ef54461cfa02571ae0db9a0dc1e0cdb5577484a6d75e68dc38e8acc1
32 B / 32 B [==============================================================] 0s
Copying config sha256:4786863027c87a85710f432101cc04157fccbe51aee9bf9de3695bf87623f7c3
302 B / 302 B [============================================================] 0s
Writing manifest to image destination
Storing signatures
4786863027c87a85710f432101cc04157fccbe51aee9bf9de3695bf87623f7c3
[sahsingh@sahsingh gophinoff]$ buildah from tm-intermediate-1
tm-intermediate-1-working-container
[sahsingh@sahsingh gophinoff]$ buildah containers
CONTAINER ID BUILDER IMAGE ID IMAGE NAME CONTAINER NAME
41b2e348f98a * scratch working-container
fb6c8f1f72c7 * 4786863027c8 localhost/tm-intermediate-1:latest tm-intermediate-1-working-container
[sahsingh@sahsingh gophinoff]$ buildah rm working-container
41b2e348f98a859821095787804d97b1d80332e7af36717e92fe7b44b4041bd5
[sahsingh@sahsingh gophinoff]$
```

So in true `docker build` fashion, we execute a step, capture that step in an intermediate layer, spawn a new working container from intermediate layer to execute the next step, and finally remove the first working container. We can repeat the process to execute the rest of the steps.

```
# Dockerfile step 2: COPY taskmaster /
[sahsingh@sahsingh gophinoff]$ buildah copy tm-intermediate-1-working-container taskmaster/taskmaster /
[sahsingh@sahsingh gophinoff]$ buildah commit tm-intermediate-1-working-container tm-intermediate-2
[sahsingh@sahsingh gophinoff]$ buildah rm tm-intermediate-1-working-container

# Dockerfile step 3: EXPOSE 8080
[sahsingh@sahsingh gophinoff]$ buildah from tm-intermediate-2
[sahsingh@sahsingh gophinoff]$ buildah config --port 8080 tm-intermediate-2-working-container
[sahsingh@sahsingh gophinoff]$ buildah commit tm-intermediate-2-working-container tm-intermediate-3
[sahsingh@sahsingh gophinoff]$ buildah rm tm-intermediate-2-working-container

# Dockerfile step 4: ENTRYPOINT [ "/taskmaster" ] 
[sahsingh@sahsingh gophinoff]$ buildah from tm-intermediate-3
[sahsingh@sahsingh gophinoff]$ buildah config --entrypoint '["/taskmaster"]' tm-intermediate-3-working-container
[sahsingh@sahsingh gophinoff]$ buildah commit tm-intermediate-3-working-container saharshsingh/taskmaster
[sahsingh@sahsingh gophinoff]$ buildah rm tm-intermediate-3-working-container
```

I removed the output of the commands above for brevity. Instead, we can look at the output of the commands below for the big picture instead.

```
[sahsingh@sahsingh gophinoff]$ buildah containers
CONTAINER ID BUILDER IMAGE ID IMAGE NAME CONTAINER NAME
[sahsingh@sahsingh gophinoff]$ buildah images
IMAGE NAME IMAGE TAG IMAGE ID CREATED AT SIZE
localhost/tm-intermediate-1 latest 4786863027c8 Jan 18, 2019 17:14 1.67 KB
localhost/tm-intermediate-2 latest 26ac01201862 Jan 18, 2019 17:22 6.88 MB
localhost/tm-intermediate-3 latest 574d0f313b8d Jan 18, 2019 17:25 6.88 MB
localhost/saharshsingh/taskmaster latest ff407f6aaa8e Jan 18, 2019 17:28 6.88 MB
```

Now this is overkill. I do not need to keep each step separated in its own image layer. This was just to illustrate the flexibility allowed by `buildah`. In practice, I probably would want to put all these steps in the same layer. I also would like to script these steps so that they become part of my build file.

{{< highlight bash "linenos=table" >}}
tm_container=$(buildah from scratch)
buildah copy $tm_container taskmaster /
buildah config --port 8080 --entrypoint '["/taskmaster"]' --label "maintainer=Saharsh Singh" $tm_container
buildah commit $tm_container saharshsingh/taskmaster
buildah rm $tm_container
{{< /highlight >}}

That does the job. Do also note how I am able to combine all my config steps in one command. Now to incorporate this into my build file, I will create a flag called `–using-buildah` that the caller can supply to build using `buildah`. By default I can have my script default to Docker.

{{< highlight bash "linenos=table" >}}
#!/bin/bash

cleanup() {

    # Remove working container from buildah
    if [ "$tm_container" != "" ]; then buildah rm $tm_container; fi

    # Go back to user directory
    popd
}

# Define script constants
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

# Process CLI args
USE_BUILDAH=0
if [ "$1" == "--using-buildah" ]; then
    USE_BUILDAH=1
fi

# Change directory to code location
set -e
pushd $SCRIPT_DIR
trap cleanup EXIT INT TERM

# Test code
go test ./...

# Build Go binary
time GOOS=linux CGO_ENABLED=0 go build

# Build container
if [ $USE_BUILDAH -eq 1 ]; then
    tm_container=$(buildah from scratch)
    buildah copy $tm_container taskmaster /
    buildah config --port 8080 --entrypoint '["/taskmaster"]' --label "maintainer=Saharsh Singh" $tm_container
    buildah commit $tm_container saharshsingh/taskmaster
else
    docker build -t saharshsingh/taskmaster $SCRIPT_DIR
fi
{{< /highlight >}}

Note how the last line that removes the working container has been moved to a `cleanup` function that is called by the `trap`. This allows the working container to be removed in case one of the intermediate buildah steps fails. Now we can run the buildah flow in the build script using the –using-buildah flag.

```
[sahsingh@sahsingh gophinoff]$ ./taskmaster/build.sh --using-buildah
~/go/src/github.com/saharshsingh/gophinoff/taskmaster ~/go/src/github.com/saharshsingh/gophinoff
?       github.com/saharshsingh/gophinoff/taskmaster    [no test files]
?       github.com/saharshsingh/gophinoff/taskmaster/http       [no test files]
ok      github.com/saharshsingh/gophinoff/taskmaster/impl       (cached)

real    0m0.067s
user    0m0.091s
sys     0m0.025s
0a54a535c63f8e3f9bc5cde935f23fc9eaca8c6149d49332b8fff770732a1dad
Getting image source signatures
Copying blob sha256:fd2ce6fd51eb18f8842e7ae325b4db767dd69d97d19a107f649ab9edafe86c79
 3.30 MiB / 3.30 MiB [======================================================] 0s
Copying config sha256:5d410baead92137dcd770421d28ad66325c4680a28f169f4a06eb43219de05c7
 356 B / 356 B [============================================================] 0s
Writing manifest to image destination
Storing signatures
5d410baead92137dcd770421d28ad66325c4680a28f169f4a06eb43219de05c7
18848d23f1fc500e84563d2139bc6a5c7a4628cd9f3c5cde1005a045723c1426
~/go/src/github.com/saharshsingh/gophinoff
[sahsingh@sahsingh gophinoff]$
```

Awesome! So now that we can build them, it’s time to run our container using Podman.

## Running Containers with Podman

Podman is pretty straight forward. For all your `docker run` needs, just replace `docker` with `podman`. So the container I just built can easily be run using `podman`.

```
[sahsingh@sahsingh gophinoff]$ podman run -d --name taskmaster saharshsingh/taskmaster
e10b1bc3c6529d2dfee6529458551cb27911e0964ccea5fdd04c13c0d2da509e
[sahsingh@sahsingh gophinoff]$ podman ps
CONTAINER ID  IMAGE                                     COMMAND      CREATED         STATUS             PORTS  NAMES
e10b1bc3c652  localhost/saharshsingh/taskmaster:latest  /taskmaster  22 seconds ago  Up 22 seconds ago         taskmaster
```

Just like `buildah`, `podman` also keeps containers user specific.

```
[sahsingh@sahsingh gophinoff]$ sudo podman ps
[sudo] password for sahsingh:
CONTAINER ID  IMAGE  COMMAND  CREATED  STATUS  PORTS  NAMES
[sahsingh@localhost gophinoff]$
```

Great! Next we would ideally want to test to make sure `podman` is actually running the container instance. However there is one minor issue. Unfortunately, you can’t do port bindings as a non root user. This means I can’t expose the port on which Task Master listens for requests outside the container.

```
[sahsingh@sahsingh gophinoff]$ podman run -d -p 8080:8080 --name taskmaster saharshsingh/taskmaster
port bindings are not yet supported by rootless containers
```

So to run my app so that it can actually be used, I need to run it via `podman` as a `root` user. Now I have two options. I can either rebuild the image as `root` using `buildah`. Or, I can do something unnecessarily complicated. Let’s do the latter. I will run a Docker registry locally exposed on port `5000`. Then I can push my image using `buildah` as `sahsingh` user to this registry, so that `podman`, run as `root`, can pull it.

```
[sahsingh@sahsingh gophinoff]$ sudo podman run -d -p 5000:5000 --name registry registry
Trying to pull docker.io/registry:latest...Getting image source signatures
Copying blob cd784148e348: 2.10 MiB / 2.10 MiB [============================] 1s
Copying blob 0ecb9b11388e: 611.83 KiB / 611.83 KiB [========================] 1s
Copying blob 45793cf0ff93: 6.51 MiB / 6.51 MiB [============================] 1s
Copying blob d7eadb9e7675: 370 B / 370 B [==================================] 1s
Copying blob 4b2356bbbed3: 213 B / 213 B [==================================] 1s
Copying config 116995fd6624: 3.09 KiB / 3.09 KiB [==========================] 0s
Writing manifest to image destination
Storing signatures
de85373e682a139aba705450a68a6d0d4fd1273d74bf228493a1d4a510e23a16
[sahsingh@sahsingh gophinoff]$ buildah tag saharshsingh/taskmaster localhost:5000/saharshsingh/taskmaster
[sahsingh@sahsingh gophinoff]$ buildah push localhost:5000/saharshsingh/taskmaster
Getting image source signatures
Copying blob sha256:2f9f39ea93e54364337f35def7b0f04e39b2c1590a988adaae5ec739841c15f6
 6.56 MiB / 6.56 MiB [======================================================] 0s
Copying config sha256:5d410baead92137dcd770421d28ad66325c4680a28f169f4a06eb43219de05c7
 356 B / 356 B [============================================================] 0s
Writing manifest to image destination
Storing signatures
Successfully pushed //localhost:5000/saharshsingh/taskmaster:latest@sha256:9a61d636eebd230812d916dc849bc79e86b51fd8ea71a5c6fca9c897ce9a953c
[sahsingh@sahsingh gophinoff]$ sudo podman pull localhost:5000/saharshsingh/taskmaster
Trying to pull localhost:5000/saharshsingh/taskmaster...Getting image source signatures
Copying blob fd2ce6fd51eb: 3.30 MiB / 3.30 MiB [============================] 0s
Copying config 5d410baead92: 356 B / 356 B [================================] 0s
Writing manifest to image destination
Storing signatures
5d410baead92137dcd770421d28ad66325c4680a28f169f4a06eb43219de05c7
```

Great! Now we can run and test the image.

```
[sahsingh@sahsingh gophinoff]$ sudo podman run -d -p 8080:8080 --name taskmaster saharshsingh/taskmaster
c98ee6cccef9a9103afad7d3958efe93a4ad085ae424e6e98e59d44901dd9488
[sahsingh@localhost gophinoff]$ curl -i localhost:8080/task && echo
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 21 Jan 2019 06:10:52 GMT
Content-Length: 4

null
[sahsingh@localhost gophinoff]$ curl -i localhost:8080/task -XPOST -H "Content-Type: application/json" -d '{"Name":"Some Task"}' && echo
HTTP/1.1 201 Created
Date: Mon, 21 Jan 2019 06:11:06 GMT
Content-Length: 0


[sahsingh@sahsingh gophinoff]$ curl -i localhost:8080/task && echo
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 21 Jan 2019 06:11:30 GMT
Content-Length: 50

{"Name":"Some Task","Description":"","Priority":0}
```

Runs just like before and all without a daemon!

## Skopeo

Finally, let’s look at `skopeo`. Skopeo is all about working with images in remote repositories – **transferring** them, **inspecting** them, and even **deleting** them. So, let’s use `skopeo` to transfer the Task Master image from the local Docker registry I am running using `podman` to my official Docker Hub account. First, let’s inspect.

```
[sahsingh@sahsingh gophinoff]$ skopeo inspect docker://localhost:5000/saharshsingh/taskmaster
{
    "Name": "localhost:5000/saharshsingh/taskmaster",
    "Digest": "sha256:9a61d636eebd230812d916dc849bc79e86b51fd8ea71a5c6fca9c897ce9a953c",
    "RepoTags": [
        "latest"
    ],
    "Created": "2019-01-21T05:31:53.86215492Z",
    "DockerVersion": "",
    "Labels": {
        "maintainer": "Saharsh Singh"
    },
    "Architecture": "amd64",
    "Os": "linux",
    "Layers": [
        "sha256:fd2ce6fd51eb18f8842e7ae325b4db767dd69d97d19a107f649ab9edafe86c79"
    ]
}
[sahsingh@sahsingh gophinoff]$ skopeo inspect docker://saharshsingh/taskmaster
FATA[0000] Error reading manifest latest in docker.io/saharshsingh/taskmaster: manifest unknown: manifest unknown
```

Seeing how I have the image at `localhost:5000` but not at Docker Hub, let’s do a copy. I have replaced my actual username and password below with a stub `user:pass`.

```
[sahsingh@sahsingh gophinoff]$ skopeo copy --dest-creds=user:pass docker://localhost:5000/saharshsingh/taskmaster docker://saharshsingh/taskmaster
Getting image source signatures
Copying blob fd2ce6fd51eb: 3.30 MiB / 3.30 MiB [============================] 2s
Copying config 5d410baead92: 356 B / 356 B [================================] 1s
Writing manifest to image destination
Copying config 5d410baead92: 0 B / 356 B [----------------------------------] 0s
Writing manifest to image destination
Storing signatures
[sahsingh@sahsingh gophinoff]$ skopeo inspect docker://saharshsingh/taskmaster
{
    "Name": "docker.io/saharshsingh/taskmaster",
    "Digest": "sha256:f4044c55bfda0ac2c74547f6fe8d7b5bbf4520974f6f342ca62cce379c1ce16f",
    "RepoTags": [
        "latest"
    ],
    "Created": "2019-01-21T05:31:53.86215492Z",
    "DockerVersion": "",
    "Labels": {
        "maintainer": "Saharsh Singh"
    },
    "Architecture": "amd64",
    "Os": "linux",
    "Layers": [
        "sha256:fd2ce6fd51eb18f8842e7ae325b4db767dd69d97d19a107f649ab9edafe86c79"
    ]
}
```

Boom!

## End

So with that this is it for this blog entry. Hope you enjoyed learning about the new set of daemonless and rootless container tooling. Till next time..
