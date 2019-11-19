---
layout: post
title:  "Writing Operators using the Operator Framework SDK"
author: "Mario"
categories: [ okd, origin, containers, kubernetes, operators, controllers, operator framework, operator sdk, openshift ]
featured: true
image: assets/images/2019-05-18-openshift-operator-framework.jpg
image-author: "Dan Lohmar"
image-author-link: "https://unsplash.com/@dlohmar"
image-source: "Unsplash"
image-source-link: "https://unsplash.com/photos/0zeb4q6odlE"
permalink: /writing-operators-using-operator-framework/
last_modified_at: "2019-11-18"
hidden: false
---

# Operators, operators everywhere

As you may have noticed, **Kubernetes operators** are becoming more an more popular those days. In this post we are going to explain the basics around Operators and we will develop a simple Operator using the **Operator Framework SDK**.

# What is an Operator

An operator aims to automate actions usually performed manually while lessening the likelihood of error and simplifying complexity.

We can think of an operator as a method of packaging, deploying and managing a Kubernetes enabled application. Kubernetes enabled applications are deployed on Kubernetes and managed using the Kubernetes APIs and tooling.

Kubernetes APIs can be extended in order to enable new types of Kubernetes enabled applications. We could say that Operators are the runtime that manages such applications.

A simple Operator would define how to deploy an application, whereas an advanced one will also take care of day-2 operations like backup, upgrades, etc.

Operators use the **Controller pattern**, but not all Controllers are Operators. We could say it's an Operator if it's got:

* Controller Pattern
* API Extension
* Single-App Focus

Feel free to read more about operators on the [Operator FAQ by CoreOS](https://coreos.com/operators/)


# Kubernetes Controllers

In the Kubernetes world, the **Controllers** take care of routine tasks to ensure cluster's observed state, matches cluster's desired state.

Each Controller is responsible for a particular resource in Kubernetes. The Controller runs a control loop that watches the shared state of the cluster through the Kubernetes API server and makes changes attempting to move the current state towards the desired state.

Some examples:

* Replication Controller
* Cronjob Controller

## Controller Components

There are two main components in a controller: `Informer/SharedInformer` and `WorkQueue`.

### Informer

In order to retrieve information about an object, the Controller sends a request to the Kubernetes API server. However, querying the API repeatedly can become expensive when dealing with thousands of objects.

On top of that, the Controller doesn't really need to send requests continuously. It only cares about CRUD events happening on the objects it's managing.

Informers are not much used in the current Kubernetes, instead SharedInformers are used.

### SharedInformer

A Informer creates a local cache for a set of resources used by itself. In Kubernetes there are multiple controllers running an caring about multiple kinds of resources though.

Having a shared cache among Controllers instead of one cache for each Controller sounds like a plan, that's a SharedInformer.

### WorkQueue

The `SharedInformer` can't track what each Controller is up to, so the Controller must provide its own queuing and retrying mechanism.

Whenever a resource changes, the SharedInformer's Event Handler puts a key into the `WorkQueue` so the Controller will take care of that change.

## How a Controller Works

### Control Loop

Every controller has a **Control Loop** which basically does:

1. Processes every single item from the WorkQueue
2. Pops an item and do whatever it needs to do with that item
3. Pushes the item back to the WorkQueue if required
4. Updates the item status to reflect the new changes
5. Starts over

**Code Examples**

* [https://github.com/kubernetes/sample-controller/blob/master/controller.go#L180](https://github.com/kubernetes/sample-controller/blob/master/controller.go#L180)
* [https://github.com/kubernetes/sample-controller/blob/master/controller.go#L187](https://github.com/kubernetes/sample-controller/blob/master/controller.go#L187)

### WorkQueue

1. Stuff is put into the WorkQueue
2. Stuff is take out from the WorkQueue in the Control Loop
3. WorkQueue doesn't store objects, it stores `MetaNamespaceKeys`

A MetaNamespaceKey is a key-value reference for an object. It has the namespace for the resource and the name for the resource.

**Code Examples**

* [https://github.com/kubernetes/sample-controller/blob/master/controller.go#L111](https://github.com/kubernetes/sample-controller/blob/master/controller.go#L111)
* [https://github.com/kubernetes/sample-controller/blob/master/controller.go#L188](https://github.com/kubernetes/sample-controller/blob/master/controller.go#L188)

### SharedInformer

As we said before, is a shared data cache which distributes the data to all the `Listers` interested in knowing about changes happening to specific objects.

The most important part of the `haredInformer` are the `EventHandlers`. Using an `EventHandler` is how you register your interest in specific object updates like addition, creation, updation or deletion.

When an update occurs, the object will be put into the WorkQueue so it gets processed by the Controller in the Control Loop.

`Listers` are an important part of the `SharedInformers` as well. `Listers` are designed specifically to be used within Controllers as they have access to the cache.

**Listers vs Client-go**

Listers have access to the cache whereas Client-go will hit the Kubernetes API server (which is expensive when dealing with thousands of objects).

**Code Examples**

* [https://github.com/kubernetes/sample-controller/blob/master/controller.go#L252](https://github.com/kubernetes/sample-controller/blob/master/controller.go#L252)
* [https://github.com/kubernetes/sample-controller/blob/master/controller.go#L274](https://github.com/kubernetes/sample-controller/blob/master/controller.go#L274)

## SyncHandler A.K.A Reconciliation Loop

The first invocation of the `SyncHandler` will always be getting the `MetaNamespaceKey` for the resource it needs to work with.

With the `MetaNamespaceKey` the object is gathered from the cache, but well.. it's not really an object, but a pointer to the cached object.

With the object reference we can read the object, in case the object needs to be updated, then the object have to be DeepCopied. `DeepCopy` is an expensive operation, making sure the object will be modified before calling `DeepCopy` is a good practice.

With the object reference / DeepCopy we are ready to apply our business logic.

**Code Examples**

* [https://github.com/kubernetes/sample-controller/blob/master/controller.go#L243](https://github.com/kubernetes/sample-controller/blob/master/controller.go#L243)

## Kubernetes Controllers

Some information about controllers:

* Cronjob controller is probably the smallest one out there
* [Sample Controller](https://github.com/kubernetes/sample-controller) will help you getting started with Kubernetes Controllers

# Writing your very first Operator using the Operator Framework SDK

We will create a very simple Operator using the [Operator Framework SDK](https://github.com/operator-framework/operator-sdk).

The Operator will be in charge of deploying a simple [GoLang application](https://github.com/mvazquezc/reverse-words).

## Requirements

At the moment of this writing the following versions were used:

* golang-1.13.4
* Operator Framework SDK v0.12.0
* Minishift v1.33.0+ba29431

## Installing the Operator Framework SDK

```sh
RELEASE_VERSION=v0.12.0
# Linux
curl -L https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu -o /usr/local/bin/operator-sdk
# macOS
curl -L https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/operator-sdk-${RELEASE_VERSION}-x86_64-apple-darwin -o /usr/local/bin/operator-sdk
# Linux / macOS
chmod +x /usr/local/bin/operator-sdk
```

## Initializing the Operator Project

First, a new new project for our Operator will be initialized.

```sh
mkdir -p ~/operators-projects/ && cd $_
operator-sdk new reverse-words-operator --repo=github.com/<github_user>/reverse-words-operator
cd reverse-words-operator
```

## Create the Operator API Types

As previously discussed, Operators extend the Kubernetes API, the API itself is organized in groups and versions. Our Operator will define a new Group, object Kind and its versioning.

In the example below we will define a new API Group called `linuxera.org`, a new object Kind `ReverseWordsApp` and its versioning `v1alpha1`.

```sh
operator-sdk add api --api-version=linuxera.org/v1alpha1 --kind=ReverseWordsApp
```

Now it's time to define the structure of our new Object. The Spec properties that we will be using are:

* `replicas`: Will be used to define the number of replicas for our application

In the Status we will use:

* `appPods`: Will track the pods associated to our current ReverseWordsApp instance

Below a link to the code for our Type:

* [reversewordsapp_types.go](https://linuxera.org/assets/post_resources/2019-05-18-openshift-operator-framework/reversewordsapp_types.go.txt)

The Types are defined within the following file:

```sh
curl -Ls https://linuxera.org/assets/post_resources/2019-05-18-openshift-operator-framework/reversewordsapp_types.go.txt -o pkg/apis/linuxera/v1alpha1/reversewordsapp_types.go
```

Replicas will be defined as an `int32` and will reference the Spec property `replicas`. For the status AppPods will be defined as a `stringList` and will reference the Status property `appPods`.

With above changes in-place we need to re-generate some boilerplate code to take into account the latest changes in our types.

```sh
operator-sdk generate k8s
```

**OpenAPI Validation**

To update the OpenAPI Validation section in the CRD definition YAML file run the following command:

> **NOTE**: This is not required for the operator to work, but it will ensure CRD types in the schema are enforced when sending CRs to the API.

```sh
operator-sdk generate openapi
```

## Add a Controller to your Operator

Now it's time to add a Controller to our Operator, this Controller will take care of our new object `ReverseWordsApp`. 

```sh
operator-sdk add controller --api-version=linuxera.org/v1alpha1 --kind=ReverseWordsApp
```

## Code your Operator business logic

An empty controller (well, not that empty) has been created into our project, now it's time to modify it so it actually deploys our application the way we want.

Our application consists of a Deployment and a Service, so our Operator will deploy the Reverse Words App as follows:

1. A Kubernetes Deployment object will be created
2. A Kubernetes Service object will be created

Below a link to the code (commented) for our Controller:

* [reversewordsapp_controller.go](https://linuxera.org/assets/post_resources/2019-05-18-openshift-operator-framework/reversewordsapp_controller.go.txt)

```
curl -Ls https://linuxera.org/assets/post_resources/2019-05-18-openshift-operator-framework/reversewordsapp_controller.go.txt -o pkg/controller/reversewordsapp/reversewordsapp_controller.go
```

## Build the Operator

We have our Operator business logic ready, so now it's time to build our Operator and deploy it onto our cluster.

First, we will build the operator and once the image is built, we will push it to the [Quay Registry](https://quay.io/).

```sh
operator-sdk build quay.io/<your_user>/reverse-words-operator:latest --image-builder <[podman, buildah, docker]>
podman push quay.io/<your_user>/reverse-words-operator:latest
```

## Deploy the Operator


1. Create a namespace for deploying and testing our operator

    ```sh
    oc create ns operator-test
    ```
2. Deploy the required RBAC

    ```sh
    oc -n operator-test create -f deploy/role.yaml
    oc -n operator-test create -f deploy/role_binding.yaml
    oc -n operator-test create -f deploy/service_account.yaml
    ```
3. Load the CRD definition onto the cluster

    ```sh
    oc -n operator-test create -f deploy/crds/linuxera_v1alpha1_reversewordsapp_crd.yaml
    ```
4. Configure the operator deployment to use your operator's image 

    ```sh
    sed -i "s|REPLACE_IMAGE|quay.io/mavazque/reverse-words-operator:latest|g" deploy/operator.yaml
    ```
5. Deploy the Operator

    ```sh
    oc -n operator-test create -f deploy/operator.yaml
    ```
6. We should see our operator pod up and running

    ```
    {"level":"info","ts":1558257945.7185602,"logger":"cmd","msg":"Go Version: go1.12.2"}
    {"level":"info","ts":1558257945.718601,"logger":"cmd","msg":"Go OS/Arch: linux/amd64"}
    {"level":"info","ts":1558257945.7186124,"logger":"cmd","msg":"Version of operator-sdk: v0.6.0"}
    {"level":"info","ts":1558257945.7190154,"logger":"leader","msg":"Trying to become the leader."}
    {"level":"info","ts":1558257945.896446,"logger":"leader","msg":"No pre-existing lock was found."}
    {"level":"info","ts":1558257945.902715,"logger":"leader","msg":"Became the leader."}
    {"level":"info","ts":1558257946.0164323,"logger":"cmd","msg":"Registering Components."}
    {"level":"info","ts":1558257946.0166807,"logger":"kubebuilder.controller","msg":"Starting EventSource","controller":"reversewordsapp-controller","source":"kind source: /, Kind="}
    {"level":"info","ts":1558257946.016886,"logger":"kubebuilder.controller","msg":"Starting EventSource","controller":"reversewordsapp-controller","source":"kind source: /, Kind="}
    {"level":"info","ts":1558257946.0170114,"logger":"kubebuilder.controller","msg":"Starting EventSource","controller":"reversewordsapp-controller","source":"kind source: /, Kind="}
    {"level":"info","ts":1558257946.1341326,"logger":"metrics","msg":"Metrics Service object created","Service.Name":"reverse-words-operator","Service.Namespace":"operator-test"}
    {"level":"info","ts":1558257946.134174,"logger":"cmd","msg":"Starting the Cmd."}
    {"level":"info","ts":1558257946.2346628,"logger":"kubebuilder.controller","msg":"Starting Controller","controller":"reversewordsapp-controller"}
    {"level":"info","ts":1558257946.3348403,"logger":"kubebuilder.controller","msg":"Starting workers","controller":"reversewordsapp-controller","worker count":1}
    ```
7. Now it's time to create ReverseWordsApp instances

    ```sh
    cp deploy/crds/linuxera_v1alpha1_reversewordsapp_cr{.yaml,2.yaml}
    vim deploy/crds/linuxera_v1alpha1_reversewordsapp_cr.yaml

    apiVersion: linuxera.org/v1alpha1
    kind: ReverseWordsApp
    metadata:
        name: example-reversewordsapp
    spec:
        replicas: 1

    vim deploy/crds/linuxera_v1alpha1_reversewordsapp_cr2.yaml

    apiVersion: linuxera.org/v1alpha1
    kind: ReverseWordsApp
    metadata:
        name: example-reversewordsapp-2
    spec:
        replicas: 2
    ```
8. And finally load them onto the cluster

    ```sh
    oc -n operator-test create -f deploy/crds/linuxera_v1alpha1_reversewordsapp_cr.yaml
    oc -n operator-test create -f deploy/crds/linuxera_v1alpha1_reversewordsapp_cr2.yaml
    ```
9. We should see two deployments and services being created, and if wee look at the status of our object we should see the pods backing the instance

    ```sh
    oc -n operator-test get reversewordsapps example-reversewordsapp -o yaml
    
    apiVersion: linuxera.org/v1alpha1
    kind: ReverseWordsApp
    metadata:
      creationTimestamp: "2019-05-19T10:08:40Z"
      generation: 1
      name: example-reversewordsapp
      namespace: operator-test
      resourceVersion: "2520014"
      selfLink: /apis/linuxera.org/v1alpha1/namespaces/operator-test/reversewordsapps/example-reversewordsapp
      uid: 13253889-7a1e-11e9-9569-0e026de60364
    spec:
      replicas: 1
    status:
      appPods:
      - deployment-example-reversewordsapp-674b4d6cbf-cpdmk

    oc -n operator-test get reversewordsapps example-reversewordsapp-2 -o yaml

    apiVersion: linuxera.org/v1alpha1
    kind: ReverseWordsApp
    metadata:
      creationTimestamp: "2019-05-19T10:08:43Z"
      generation: 1
      name: example-reversewordsapp-2
      namespace: operator-test
      resourceVersion: "2520074"
      selfLink: /apis/linuxera.org/v1alpha1/namespaces/operator-test/reversewordsapps/example-reversewordsapp-2
      uid: 153c796d-7a1e-11e9-9569-0e026de60364
    spec:
      replicas: 2
    status:
      appPods:
      - deployment-example-reversewordsapp-2-5654fcddd6-25qpt
      - deployment-example-reversewordsapp-2-5654fcddd6-znwzw  
    ```
10. We can test our application now

    ```sh
    LB_ENDPOINT=$(oc -n operator-test get svc --selector='app=example-reversewordsapp' -o jsonpath='{.items[*].status.loadBalancer.ingress[*].hostname}')
    
    curl -X POST -d '{"word":"PALC"}' http://$LB_ENDPOINT:8080
    {"reverse_word":"CLAP"}
    ```
11. Cleanup

    ```sh
    oc delete -f deploy/crds/linuxera_v1alpha1_reversewordsapp_cr.yaml
    oc delete -f deploy/crds/linuxera_v1alpha1_reversewordsapp_cr2.yaml
    oc delete -f deploy/operator.yaml
    oc delete -f deploy/crds/linuxera_v1alpha1_reversewordsapp_crd.yaml
    oc delete ns operator-test
    ```
12. That's it!

# In the next episode:

* Readiness and liveness probes will be added to our Deployments
* A new property `Release` will be added to our object

# Sources

* [Operators by CoreOS](https://coreos.com/operators/)
* [A deep dive into Kubernetes Controllers](https://engineering.bitnami.com/articles/a-deep-dive-into-kubernetes-controllers.html)
* [Writing Kube Controllers for Everyone](https://www.youtube.com/watch?v=AUNPLQVxvmw)
