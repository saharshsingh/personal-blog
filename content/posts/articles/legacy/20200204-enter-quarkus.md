+++
author = "Saharsh Singh"
title = "Enter Quarkus"
slug= "enter-quarkus"
date = "2020-02-04"
description = "In this article, I explore the basics of Quarkus, a brand new Kubernetes Native Java framework built to specifically address Java’s weight problem."
tags = [
    "code",
    "containerization",
    "frameworks",
    "howto",
    "java",
    "kubernetes",
    "software",
    "tech",
    "work",
]
categories = [
    "articles",
]
aliases = [
    "/2020/02/04/enter-quarkus/",
]
+++

Java has been in a bit of an awkward spot since containers took off a few years ago. In the world of Kubernetes, microservices, and serverless, it has been getting harder and harder to ignore that Java applications are, by today’s standards, overweight. Well, until now. In this article, I explore the basics of Quarkus, a brand new Kubernetes Native Java framework built to specifically address Java’s weight problem.

<!--more-->

## Java of Yore

For years, many of us have looked the other way when confronted with the bloatiness of Java. Who cares if my server side app needed **hundreds of megabytes** worth of **class files**, created **gigabytes** worth of **runtime memory footprint**, and took up to a **minute (maybe five) to start up**. I definitely didn’t dare fat shame Java as long as, after the slow start of my obese deployment, my application instance could then run reliably on a powerful piece of hardware or VM for months, if not years, serving hundreds of requests concurrently. Not to mention, as a language, Java gave me pretty much everything I needed – a “write once, run (almost) anywhere” platform in the JVM, type safety, OOP support, and unrivaled set of options in tooling and libraries – to maintain my software for a long time using a large team of professionals with varying levels of skills. Further, with **enterprise grade application servers** (e.g. EAP, Weblogic, Tomcat), you also had a very resilient and feature rich platform for your Java web applications. Your applications simply needed to comply with JavaEE standards around describing how to deploy your application (think web.xml). Any JavaEE compliant application server would then take care of operational concerns like security, logging, connecting to databases/queues, and scaling. It’s no surprise that for years **Java** has **dominated** the programming language landscape as **the de facto standard** for the enterprise.

## Kubernetes: The New Application Server

It has been observed [before](https://developers.redhat.com/blog/2018/06/28/why-kubernetes-is-the-new-application-server/), by more people than one, that Kubernetes is the new application server. Containers and Kubernetes have taken the “write once run anywhere” paradigm of the JVM and extended it to pretty much all other programming languages. Now applications written in any language can leverage Kubernetes for operational concerns and decouple themselves from runtime infrastructure. The applications just have to be delivered in compliant Linux containers. Now developers can code up their applications in their favorite programming language and count on Kubernetes to handle operational concerns like logging, scaling, healing, and networking. Add in Istio and you even have out of box fault tolerance and application level metrics without a single line of application code.

So, today we find ourselves in a tech landscape that overwhelmingly prefers [horizontal scaling of automated cattle over vertical scaling of manually cared for pets](http://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/). Microservices and serverless/FaaS applications have become all the rage and both benefit greatly from low memory footprint and blazing fast startup times. So, it has become increasingly harder to ignore that my Java container images are quite a bit larger in size as well as memory footprint, and they take quite a bit longer to start up, especially when compared to a language like Golang. Modern cloud native frameworks like Spring Boot or Dropwizard have helped, but startup times are still AT LEAST ten seconds or more, and runtime memory footprint is AT LEAST in hundreds of megabytes.

## Enter Quarkus

**Quarkus** aims to tackle the **bloatiness problem of Java** head on. Marketed as *Supersonic Subatomic Java*, Quarkus leverages [GraalVM](https://www.graalvm.org/) and [HotSpot](http://openjdk.java.net/groups/hotspot/) to provide developers with a framework to create applications from Java code with fast boot time and low RSS memory. The following graphic from [quarkus.io](https://quarkus.io/) does a good job illustrating the benefits. Notice the drastic difference in both RSS memory and boot time between Quarkus native and “the traditional cloud native stack”.

{{< figure class="img-responsive" src="http://saharsh.org/wp-content/uploads/2020/02/quarkus_metrics_graphic_bootmem_wide-768x355.png" >}}

## OpenJDK and GraalVM

As evident from the graphic above, Quarkus has two modes – **JVM** and **native**. The native mode uses GraalVM to create a [standalone executable](https://www.graalvm.org/docs/reference-manual/native-image/) that doesn’t run in a Java VM. Obviously, the greatest efficiency gains come from running a Quarkus application in its native mode. However, not every JVM feature works in native mode, and the most notorious of these lost features is [reflection](https://www.oracle.com/technical-resources/articles/java/javareflection.html). This can be a huge problem as a great number of frameworks and libraries that Java developers have depended on for every day development rely heavily on reflection. GraalVM works around this by allowing classes to be registered for reflection at compile time. While this process can be cumbersome when working directly with GraalVM, Quarkus streamlines the registration process by detecting and auto-registering as many of your code’s reflection candidates as possible.

While Quarkus has done a pretty good job with auto registering most reflection candidates you are likely to have, you might still run into instances where you have to explicitly register some of your classes using Quarkus’s `RegisterForReflection` [annotation](https://javadoc.io/doc/io.quarkus/quarkus-core/latest/io/quarkus/runtime/annotations/RegisterForReflection.html). This might become more trouble than worth in some projects. For this reason as well as just general flexibility, Quarkus also offers the JVM mode. In JVM mode, Quarkus apps are packaged as JAR files and run on the OpenJDK HotSpot JVM.

## Show me the Code!

So having set the stage, let’s look at some code. To get started with Quarkus, I put together a JAX-RS application following the excellent [“Getting Started” guides from Quarkus](https://quarkus.io/guides/getting-started).  See [my repo](https://github.com/saharsh-samples/sample-quarkus-app) for the application code. The application is a simple service that can be used to store, update, retrieve, and delete arbitrary text values. I mostly just followed the guide as I wrote my code. I built the application out in the following stages.

### Core Application

In this stage, I created the core application with all the API endpoints. I started by generating an app skeleton using the `quarkus-maven-plugin` and adding the `resteasy-jackson` extension for JSON support.

{{< highlight bash "linenos=table" >}}
mvn io.quarkus:quarkus-maven-plugin:1.2.0.Final:create \
    -DprojectGroupId=org.saharsh \
    -DprojectArtifactId=sample-quarkus-app \
    -DclassName="org.saharsh.samples.quarkus.resources.ValuesResource" \
    -Dpath="/api/values"

mvn quarkus:add-extension -Dextensions="resteasy-jackson"
{{< /highlight >}}

Some changes I made was getting rid of `.dockerignore` and the `Dockerfile` examples generated by the `create` task of the `quarkus-maven-plugin`. Instead I prefer to use a [multi-stage Dockerfile](https://docs.docker.com/develop/develop-images/multistage-build/) (see [JVM](https://github.com/saharsh-samples/sample-quarkus-app/blob/master/Dockerfile) and [Native](https://github.com/saharsh-samples/sample-quarkus-app/blob/master/Dockerfile.native)) to keep my build concerns in one file. After this it was just adding my application code as captured in this [tag](https://github.com/saharsh-samples/sample-quarkus-app/releases/tag/core-app) (or [commit](https://github.com/saharsh-samples/sample-quarkus-app/commit/b087b495cee5c35ea7e8072fe2bb4d08a2dd29f5)).

### Metrics and Healthchecks

Metrics and health checks are crucial in creating twelve-factor applications. Quarkus, leveraging [Microprofile](https://projects.eclipse.org/projects/technology.microprofile), makes adding these in very straightforward.

        mvn quarkus:add-extension -Dextensions="metrics"

See this [tag](https://github.com/saharsh-samples/sample-quarkus-app/releases/tag/metrics) (and [commit](https://github.com/saharsh-samples/sample-quarkus-app/releases/tag/metrics)) for the metrics I added for the application. Essentially my application collects timing metrics for all its exposed API endpoints. It also has a gauge on the value store’s size. Metrics are published at the `/metrics` endpoint, which contains `base`, `vendor`, and `application` metrics. Each one of those subgroups also has its own endpoint (e.g. `/metrics/application`).

        mvn quarkus:add-extension -Dextensions="health"

Similarly, see this [tag](https://github.com/saharsh-samples/sample-quarkus-app/releases/tag/healthchecks) (and [commit](https://github.com/saharsh-samples/sample-quarkus-app/commit/784451eb13fb3aedffc433270dfeedf57abb24dc)) for the healthchecks. I added a [liveness check](https://github.com/saharsh-samples/sample-quarkus-app/blob/master/src/main/java/org/saharsh/samples/quarkus/health/LivenessCheck.java) and a [readiness check](https://github.com/saharsh-samples/sample-quarkus-app/blob/master/src/main/java/org/saharsh/samples/quarkus/health/ReadinessCheck.java). The `/health` endpoint can be accessed for all healthchecks aggregated in one. However, you typically separate these into liveness and readiness probes. For this reason, `/health/live` and `/health/ready` endpoints are also automatically provided.

### Persistence

The core app I put together in the first stage uses an in memory storage service. This means the storage is local to each instance of the application and gets wiped when that instance goes down. To build an actual stateless application that can be scaled up and have persistent storage, let’s offload the application state to a MySQL database.

        mvn quarkus:add-extension -Dextensions="hibernate-orm,jdbc-mysql"

See this [tag](https://github.com/saharsh-samples/sample-quarkus-app/releases/tag/persistent-storage) (and [commit](https://github.com/saharsh-samples/sample-quarkus-app/commit/6912d349c827ceb28b8c5424af7f924d9dd03a93)) for changes related to persistence. Highlights are below:

1. Having [my resource class](https://github.com/saharsh-samples/sample-quarkus-app/blob/master/src/main/java/org/saharsh/samples/quarkus/resources/ValuesResource.java) depend on the [StorageService](https://github.com/saharsh-samples/sample-quarkus-app/blob/master/src/main/java/org/saharsh/samples/quarkus/resources/ValuesResource.java) interface abstraction, I have to make zero code changes there to switch to persistent mode.

1. To be able to pick my storage service implementation at runtime I introduce three things:

    * A `sample.storage.type` property

    * A [producer class](https://github.com/saharsh-samples/sample-quarkus-app/blob/master/src/main/java/org/saharsh/samples/quarkus/service/StorageServiceProducer.java) to create the right bean based on the property

    * A [Qualifier](https://javaee.github.io/javaee-spec/javadocs/javax/inject/Qualifier.html) annotation ([ConfiguredStorage](https://github.com/saharsh-samples/sample-quarkus-app/blob/master/src/main/java/org/saharsh/samples/quarkus/service/ConfiguredStorage.java)) for my resource class to specify that it intends to use the bean produced by the producer class.

1. I leverage the [`application.properties`](https://github.com/saharsh-samples/sample-quarkus-app/blob/master/src/main/resources/application.properties) pattern to use in-memory storage as the default storage type. I intend to use [environment variables to override these properties](https://quarkus.io/guides/config#overriding-properties-at-runtime) to switch over to persistent storage. There is one gotcha here however. Quarkus does much of its configuration and bootstrap at build time. Most properties will then be read and set during the build time step. To change them, make sure to repackage your application. In my `application.properties` file, `quarkus.hibernate-orm.dialect`, `quarkus.datasource.driver`, and `quarkus.datasource.health`.enabled cannot be overridden at runtime. Good news is that the rest can.

### And that’s it

I have a couple more commits around adding native build support and documentation. However, the application is ready to go. My repo `README.md` does a good job of walking through the details of building and running this application locally. You can use the following steps as a reference for running the application on Openshift or Code Ready Containers.

{{< highlight bash "linenos=table" >}}
# Create a new project
oc new-project samples

# Standup MySQL
oc new-app --name=valuesdb  mysql-ephemeral \
  -p DATABASE_SERVICE_NAME=valuesdb \
  -p MYSQL_ROOT_PASSWORD=password \
  -p MYSQL_USER=valsuser \
  -p MYSQL_PASSWORD=password \
  -p MYSQL_DATABASE=valsdb

# Create the application schema in MySQL
oc rsh valuesdb-1-[pod_id] bash -c "mysql -uvalsuser -ppassword valsdb"

mysql> CREATE TABLE vals (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  value VARCHAR(255) NOT NULL,
  date_created TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

# Create API application from Github repo
oc new-app --name valuesapi https://github.com/saharsh-samples/sample-quarkus-app

# Expose a route
oc expose svc/valuesapi && oc get routes

# To use persistent storage, first create a secret containing DB configuration
oc create secret generic valuesapi-properties \
  --from-literal=SAMPLE_STORAGE_TYPE=persistent \
  --from-literal=QUARKUS_DATASOURCE_URL="jdbc:mysql://valuesdb/valsdb" \
  --from-literal=QUARKUS_DATASOURCE_USERNAME=valsuser \
  --from-literal=QUARKUS_DATASOURCE_PASSWORD=password

# Turn the fields of the secret into environment variables for the API app
oc set env dc/valuesapi --from=secret/valuesapi-properties

# Add liveness and readiness probes
oc set probe dc/valuesapi --liveness --get-url=http://:8080/health/live
oc set probe dc/valuesapi --readiness --get-url=http://:8080/health/ready
{{< /highlight >}}

## Conclusion

Quarkus is an exciting new development in the Java ecosystem. I will make sure to share more articles and code as I explore Quarkus in relation to serverless, reactive programming, and Kafka. In the meanwhile, checkout the following links to dig deeper now.

* [Official website](https://quarkus.io/)
* [Official Github](https://github.com/quarkusio/quarkus)
* [Quarkus FAQ](https://quarkus.io/faq/)
* [Why GraalVM?](https://www.graalvm.org/docs/why-graal/)

