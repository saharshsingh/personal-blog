+++
author = "Saharsh Singh"
title = "Hands On Introduction to OpenShift Serverless"
slug= "openshift-serverless-intro"
date = "2020-03-02"
description = "In this article, I build upon a previous YouTube video and my recent work with Quarkus to create a hands-on deep dive into Openshift Serverless."
tags = [
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
    "/2020/03/02/hands-on-introduction-to-openshift-serverless/",
]
+++

I recently collaborated with fellow Red Hatters to create a whiteboarding video that introduces Openshift Serverless at a high level. In this article, I build upon that YouTube video and my recent work with Quarkus to create a hands-on deep dive into Openshift Serverless. This article walks you through using the Openshift Serverless operator to seamlessly add serverless capabilities to an Openshift 4.3 cluster and then using the Knative CLI tool to deploy a Quarkus native application as a serverless service onto that same cluster.

<!--more-->

## Openshift Serverless

OpenShift Serverless helps developers to deploy and run applications that will scale up or scale to zero on-demand. Applications are packaged as OCI compliant Linux containers that can be run anywhere. Using the Serverless model, an application can simply consume compute resources and automatically scale up or down based on use. As mentioned in the introduction above, the whiteboarding YouTube video embedded below provides a high level overview of Openshift Serverless.

{{< youtube DYhK60vkDQY >}}

Openshift Serverless is based on **Knative**, an open source project started by Google. Specifically, Openshift Serverless uses the **Serving** component of Knative. **Knative Serving** extends Kubernetes using Custom Resource Definitions (CRDs) to support deploying and serving of serverless applications and functions. Knative Serving is used to run containerized applications with Knative abstracting away details like networking, autoscaling (including scaling down to zero), and revision tracking. As I will demonstrate soon below, Knative Serving is easy to get started with and scales to support advanced scenarios. [Knative documentation](https://knative.dev/docs/serving/) does a good job of breaking down the components of Knative Serving and providing examples. In this article I will focus on installing Openshift Serverless and deploying [my sample Quarkus application](https://github.com/saharsh-samples/sample-quarkus-app) as a Knative Serving application.

## Installing Openshift Serverless

Best way to add serverless capabilities to an Openshift cluster is by installing the **Openshift Serverless Operator**. Adding the operator to an Openshift 4.3 cluster is straightforward.

1. Login to the web console as a cluster administrator.

1. Make sure you are using the web console in the **Administrator** perspective.

1. Under the **openshift-operators** project, navigate to the **Operators -> Operator Hub** menu item.

1. Search for **Openshift Serverless** in the “Filter by keyword” search box.
{{< figure class="img-responsive" src="/images/posts/openshift-serverless-intro/01.png" >}}

1. Click on **Openshift Serverless Operator** to start installing. Install with all the default options selected.
{{< figure class="img-responsive" src="/images/posts/openshift-serverless-intro/02.png" >}}

1. After clicking through the installation views, you’ll end up at the **Installed Operators** view. Openshift Serverless Operator depends on **Red Hat Openshift Service Mesh**, which in turn depends on **Elasticsearch**, **Jaeger**, and **Kiali**. Wait till the **Status** column for all operators has a green check mark indicating **InstallSucceeded**.
{{< figure class="img-responsive" src="/images/posts/openshift-serverless-intro/04.png" >}}

1. Create a new project called **knative-serving**. This is where the Knative Serving object that manages all serverless applications on your cluster will live.
{{< figure class="img-responsive" src="/images/posts/openshift-serverless-intro/05.png" >}}

1. Navigate to the **Operators -> Installed Operators** view under the **knative-serving** project. You’ll notice all the operators we just installed in openshift-operators project get copied here.

1. Once all the operators have a green check mark in the **Status** column indicating **Copied**, click on the **Openshift Serverless Operator.**
{{< figure class="img-responsive" src="/images/posts/openshift-serverless-intro/06.png" >}}

1. In the **Operator Details** view, click on the **Knative Serving** tab link. If the view shows a 404 page, just refresh after a few seconds.

1. Click on the **Create KnativeServing** button. In the **Create KnativeServing** form view, click **Create** to create the Serving object using the default out-of-box YAML file.
{{< figure class="img-responsive" src="/images/posts/openshift-serverless-intro/08.png" >}}

1. After creating the Knative Serving object, navigate to the **Workloads -> Pods** view and wait till all pods are in Running state.
{{< figure class="img-responsive" src="/images/posts/openshift-serverless-intro/09.png" >}}

And that’s it. You’re now ready to start deploying serverless applications to your cluster. So let’s do just that.

## Deploying a Serverless Application

I recently created [a reference Quarkus application](https://github.com/saharsh-samples/sample-quarkus-app) as part of [a blog article I wrote introducing Quarkus](http://saharsh.org/2020/02/04/enter-quarkus/). I’ll use that application as a reference here as well while demonstrating how to deploy a serverless application to OpenShift. The application is a RESTful API used to store simple text values. The values can be either stored in memory or in a MySQL database. In our serverless deployment, we will use the MySQL configuration so that the values are persistent beyond any single serverless instance’s lifetime.

To get started, let’s create a project and deploy the MySQL instance. As mentioned in the last section, serverless features are available to any user on your cluster, including ones without cluster administrator privileges. So feel free to run the following steps as a non-administrator user.

NOTE: Following steps assume familiarity with OpenShift CLI. See [OpenShift docs](https://docs.openshift.com/container-platform/4.3/cli_reference/openshift_cli/getting-started-cli.html) for an introduction.

{{< highlight bash >}}
# Create a new project
oc new-project samples

# Standup MySQL
oc new-app --name=valuesdb  mysql-ephemeral \
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
{{< /highlight >}}

NOTE: one of the steps above uses oc rsh to run the MySQL command line client inside the MySQL pod. To get the pod ID mentioned in the step, run oc get pods, and make sure the MySQL pod is running before running the command.

We will configure our serverless application to connect to the MySQL instance instead of using an in-memory value store by injecting certain environment variables into the pod. We will inject those environment variables from an OpenShift secret. Let’s create that secret now.

{{< highlight bash >}}
oc create secret generic valuesapi-properties \
  --from-literal=SAMPLE_STORAGE_TYPE=persistent \
  --from-literal=QUARKUS_DATASOURCE_URL="jdbc:mysql://valuesdb/valsdb" \
  --from-literal=QUARKUS_DATASOURCE_USERNAME=valsuser \
  --from-literal=QUARKUS_DATASOURCE_PASSWORD=password
{{< /highlight >}}

Next, we need to build and publish our application’s container image into a registry visible to our OpenShift cluster. I published a public image to my Quay registry using the following commands. Feel free to deploy to a container registry of your choice.

{{< highlight bash >}}
git clone git@github.com:saharsh-samples/sample-quarkus-app.git

podman build \
  -t quay.io/sahsingh/sample-quarkus-app:1.0 \
  -f Dockerfile.native \
  sample-quarkus-app

podman push \
  --creds sahsingh \
  quay.io/sahsingh/sample-quarkus-app:1.0
{{< /highlight >}}

We are finally ready to deploy our serverless application. The most seamless approach here is using the [Knative CLI tool](https://mirror.openshift.com/pub/openshift-v4/clients/serverless/latest/). The kn tool uses the [Kubernetes authentication configuration](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/#context) stored in the **kubeconfig** file. The `oc login` and `oc new-project` commands configure this file properly. So, the following command deploys the application as a Knative service in the `samples` project.

{{< highlight bash >}}
kn service create valuesapi \
  --image quay.io/sahsingh/sample-quarkus-app:1.0 \
  --env-from secret:valuesapi-properties
{{< /highlight >}}

Running this command creates the Knative objects needed to deploy this application as an OpenShift Serverless service. The results of this command should be similar to the following output.

```
Creating service 'valuesapi' in namespace 'samples':

  0.198s The Route is still working to reflect the latest desired specification.
  0.288s Configuration "valuesapi" is waiting for a Revision to become ready.
  9.927s ...
  9.928s Ingress has not yet been reconciled.
  9.991s Configuration "valuesapi" is waiting for a Revision to become ready.
10.111s Ingress has not yet been reconciled.
50.990s Ready to serve.

Service 'valuesapi' created with latest revision 'valuesapi-dqpmz-1' and URL:
http://valuesapi.samples.apps.cluster-plano-dc47.plano-dc47.example.opentlc.com
```

There are three OpenShift Serverless objects created: a service, a revision, and a route. You can view these objects in the OpenShift web console by navigating to **Serverless** under the **samples** project.
{{< figure class="img-responsive" src="/images/posts/openshift-serverless-intro/10.png" >}}

## Autoscaling

Going to **Workloads -> Deployments**, you’ll notice that there is a deployment associated with `valuesapi` service. However, since no requests have been sent to the application yet, there are no pods running for the deployment. This is an example of **the scaled down to zero** feature of OpenShift Serverless. Sending requests to the service’s route will trigger OpenShift Serverless to automatically scale the deployment to one pod (or more depending on the volume of requests). On my cluster, the auto generated route for the `valuesapi` service is http://valuesapi.samples.apps.cluster-plano-dc47.plano-dc47.example.opentlc.com. Similarly, once OpenShift Serverless detects that requests are no longer coming to the `valuesapi` route, the deployment is automatically scaled down to zero. The [README](https://github.com/saharsh-samples/sample-quarkus-app/blob/master/README.md#consume) of the application Git repository lists all the endpoints exposed by the valuesapi. I’ll leave it as an exercise to the readers to consume the various endpoints and test the autoscaling behavior of OpenShift Serverless.

## Conclusion

Hopefully this article demonstrates the value of OpenShift Serverless as a developer friendly platform for cloud native applications. The approach to deployment here is fairly manual and meant to primarily serve an educational purpose. For production ready applications, I encourage users of OpenShift Serverless to adopt a more automated and continuous approach using tools like [Tekton](https://tekton.dev/) or [Jenkins X](https://jenkins-x.io/).
