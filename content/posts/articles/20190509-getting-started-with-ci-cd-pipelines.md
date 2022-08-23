+++
author = "Saharsh Singh"
title = "Getting Started with CI/CD Pipelines"
slug= "getting-started-with-ci-cd-pipelines"
date = "2019-05-09"
description = "This article focuses on building CI/CD pipelines using Jenkins, Kubernetes/Openshift, Buildah, and Helm."
tags = [
    "containerization",
    "devops",
    "kubernetes",
    "sdlc",
    "software",
    "tech",
    "work",
]
categories = [
    "articles",
]
aliases = [
    "/2019/05/09/getting-stated-with-ci-cd-pipelines/",
]
+++

DevOps is becoming a ubiquitous practice among modern software organizations, and a critical aspect of DevOps are CI/CD pipelines (continuous integration, delivery, and deployment). This article focuses on building CI/CD pipelines using Jenkins, Kubernetes/Openshift, Buildah, and Helm. I will build two CI/CD pipelines. Both build, deploy, and release a sample web application. One deploys the application to any Kubernetes cluster using a Gitflow branching model, while the other deploys specifically to Openshift (Red Hat’s managed Kubernetes offering) and uses Trunk development branching model. What the hell does all this mean? Let’s dig in.

<!--more-->

## Background

As a software developer and consultant, I get to be part of digital transformation efforts of some pretty cool companies spanning a diverse set of industries. In the last few years, I have helped several companies move their applications and workloads into the cloud (typically Kubernetes) and adopt a faster, more iterative approach to software development and release. While this involves addressing various application development concerns (not to mention company culture and politics), one steady constant is building CI/CD pipelines for applications being migrated. Having done this a few times over, I wanted to capture the basic proof of concept behind these pipelines in interest of faster starts in future endeavors.

CI/CD pipelines are a staple in helping organizations achieve an iterative approach to development and a leaner, more reliable release process. On the business side, companies achieve faster time to market and deliver features their customers actually want. This article looks at building some simple pipelines that can help teams get started.

Here are the two pipelines at a high level:

### Pipeline 1

* Github repo: [https://github.com/saharsh-samples/dotnet-k8s-helm-cicd](https://github.com/saharsh-samples/dotnet-k8s-helm-cicd)
* .NET application
* Gitflow branching model
* Deployed to any Kubernetes cluster
* Container image built using Buildah on a dynamic Jenkins slave
* Container image delivered to Docker Hub (external container registry) for storage
* Deployed using Helm

### Pipeline 2

* Github repo: [https://github.com/saharsh-samples/dotnet-ocp-helm-cicd](https://github.com/saharsh-samples/dotnet-ocp-helm-cicd)
* .NET application
* Trunk development branching model
* Deployed to Openshift
* Container image built using Openshift build configuration
* Container image delivered to Openshift image streams
* Deployed using Helm and Openshift deployment configuration triggers

Alright, let’s dig deeper.

## The Tech Stack

The tech stack behind these pipelines is made up of following popular open source technologies that I’ve seen fit the needs of many organizations:

* Jenkins
* Kubernetes / Openshift
* Buildah
* Helm

**Jenkins** has pretty much been the de facto standard in continuous integration tooling for many years now. Specifically of note, Jenkins offers **declarative multibranch pipelines** and a **Kubernetes plugin** for dynamic slaves. Under multibranch pipelines, Jenkins allows you to execute and manage CI/CD pipelines for source code repositories containing multiple branches. Multibranch pipeline projects in Jenkins can be configured to either poll or receive push notifications from source code repositories (e.g. Github) regarding changes. Upon detection of changes, the appropriate branch specific pipeline can then be executed. The Kubernetes plugin allows you to leverage your existing Kubernetes cluster(s) to keep the Jenkins footprint lean and flexible. Now both, Jenkins master and slave nodes, can be run as container instances in your Kubernetes clusters. Further, slaves can be spun up only when needed and torn down when they go idle. In the pipelines presented in this article, the **Jenkinsfile** itself defines the **Kubernetes pod templates** for slaves needed to execute its various stages, relieving the need to setup and spawn any slaves directly in Jenkins prior to pipeline execution.

The Jenkins Kubernetes plugin works very well for our case since the intention is to deploy our sample application to Kubernetes clusters anyways. The first pipeline covered can be setup on any Kubernetes cluster(s). The second pipeline uses features specific to **Openshift**, Red Hat’s managed Kubernetes offering. The second pipeline showcases how Openshift can simplify concerns like container image management, release promotion, and execution of CI/CD pipeline workloads. I used **Minishift** to run both examples since I am working entirely out of my laptop, and Minishift makes it very trivial to setup a Jenkins instance.

Finally, **Buildah** and **Helm** are tools to build, package, and deploy the application into Kubernetes. **Buildah** is a daemonless alternative to `docker build` that I have covered in detail before. Using Docker to build container images means either setting up a **docker-in-docker** pod or mounting the Docker socket of the Kubernetes node into the pod. Neither of these approaches are great. Buildah solves the ‘daemon’ issue and is a great lightweight option for building images. Further, it should be soon possible to run Buildah reliably without needing the pod to be privileged in Kubernetes (i.e. rootless). Next up we have Helm. **Helm** is a popular tool for managing deployments of Kubernetes applications and is quickly becoming a de facto standard. While Buildah is used to build container images, Helm makes it easy to develop and maintain the YAML deployment descriptors used to deploy those images into Kubernetes. I am somewhat new to Helm, but have already used it on multiple projects this year and have loved its undeniable mix of simplicity and power.

## The Application

Both pipelines are built for the same application. The application really doesn’t matter and is more of a means to showcase the pipelines. It is a very basic **ASP.NET 2.2 WebAPI** project that contains an example RESTful API with the following endpoints:

* **GET** `/api/values`
* **POST** `/api/values` (accepts a JSON string in request body)
* **GET** `/api/values/{id}`
* **PUT** `/api/values/{id}` (accepts a JSON string in request body)
* **DELETE** `/api/values/{id}`

The API also comes with the following administrative endpoints:

* **GET** `/info` (metadata about the app, i.e. name, description, and version)
* **GET** `/health` (basic ASP.NET Web API healthcheck)
* **GET** `/metrics` (Prometheus compatible metrics)

The application is built using a **multi stage Dockerfile** build. The first stage uses the `microsoft/dotnet:2.2-sdk` to build .NET binaries. The second stage packages those binaries into the `microsoft/dotnet:2.2-aspnetcore-runtime` base image to produce the application’s deployable container image. A typical real world application would mature this build process to also include unit testing, integration testing, and static code analysis as part of the build.

The source code for the application can be found in both repositories under the `sample-dotnet-app` directory.

## Branching model

Most software organizations today are using one of two branching models for their source code: **Gitflow** or **Trunk Development**. I personally prefer trunk development since, when done right, it better enables a culture of iterative software development with always ready to release applications. However, I’ve seen Gitflow work well for some organizations as well. To cover both, each pipeline uses a different branching model.

First pipeline (**dotnet-k8s-helm-cicd**) uses Gitflow. The main branch is `develop`. All feature branches are created from this branch. Upon completion, the feature branches are merged back into the `develop` branch and deleted. The `develop` branch is occasionally merged into the `master` branch to create release candidates. The `master` branch is never merged into `develop`. Instead, if fixes need to be done for a release, then a branch should be created from `master`, and any changes should be both **merged into `master`** and **cherry-picked back into `develop`**. The second pipeline (**dotnet-ocp-helm-cicd**) uses Trunk Development. The main branch is `master`. All feature branches are created from this branch. Upon completion, the feature branches are merged back into the `master` branch and deleted. Essentially, in both models, commits to `master` kick off the release process. In Trunk development, since features are merged directly into `master`, releases are frequent. In Gitflow, releases are more controlled as features are merged into `develop`, a gatekeeper branch, and later merged into `master` in batches, typically on a fixed cadence (e.g. sprints).

Both pipelines automate the resulting versioning related tasks. The application version is maintained in in the `appVersion` field of the `Chart.yaml` file used for Helm installs. The version is expected to follow typical semantic versioning. When triggered by commits to `master`, the pipeline tags the `HEAD` of the `master` branch with the `appVersion` value and pushes the tag to the origin repository. In the Gitflow model, the pipeline then checks out the current `HEAD` of the `develop` branch, increments the third digit of the semantic version by one, and pushes the new `HEAD` of `develop` to the origin repository. In Trunk Development, the minor version is incremented on `master` branch directly.

All of this has implications for the deployment process as well. Both pipelines deploy to three shared environments – **Development**, **QA**, and **Production**. In Gitflow model, commits to `develop` branch automatically trigger deployments to **Development** environment. However, since commits to `develop` are not part of the release process, the pipeline goes no further. Instead, commits to `master` start the release process and trigger automatic deployments to **QA** environment. The pipeline then waits for manual confirmation that the release is healthy and can be promoted to **Production**. In this design, the latest `HEAD` of `develop` is always deployed in the shared **Development** environment. All release candidates, on the other hand, begin their journey at the **QA** environment. In Trunk Development model, the deployment process is largely the same except for one key difference. Since features get merged directly to `master` branch, commits to `master` trigger automatic deployments to both **Development** and **QA** environments.

## Pipelines in Action

I have included exhaustive documentation is both Github repositories that describes both pipelines in detail and walks through configuration and execution on Minishift. These are starting points and will need to be discussed and modified to fit the use cases of specific organizations. However, the hope is these pipelines can help aid in conversations around real world CI/CD pipelines as well as jump start their design and development. Feel free to visit and use each repository for more hands on documentation.

* [Gitflow model with Kubernetes](https://github.com/saharsh-samples/dotnet-k8s-helm-cicd)
* [Trunk Development model with Openshift](https://github.com/saharsh-samples/dotnet-ocp-helm-cicd)

