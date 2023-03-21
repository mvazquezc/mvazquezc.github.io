---
title:  "Integrating our Operators with OLM"
author: "Mario"
tags: [ "okd", "origin", "containers", "kubernetes", "operators", "controllers", "operator framework", "operator sdk", "operator lifecycle manager", "olm" ]
url: "/integrating-operators-olm/"
draft: false
date: 2020-09-16
lastmod: 2023-02-13
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Introduction

This post is a continuation of our previous blog [Writing Operators using the Operator Framework SDK](https://linuxera.org/writing-operators-using-operator-framework/).

We will continue working on the operator created on the previous blog, if you want to be able to follow this blog, you will need to run the steps from the previous blog.

# Operator Lifecycle Manager

The [Operator Lifecycle Manager](https://github.com/operator-framework/operator-lifecycle-manager) is an open source toolkit to manage Operators in an effective, automated 
and scalable way. You can learn more [here](https://github.com/operator-framework/operator-lifecycle-manager#overview).

During this post we will integrate our Reverse Words Operator into OLM, that way we will be able to use OLM to manager our Operator lifecycle.

# Integrating Reverse Words Operator into OLM

## Deploy OLM

Some Kubernetes distributions, like [OpenShift](https://www.openshift.com/) come with OLM pre-installed, if you're using a distribution with doesn't come with OLM or you don't
know if your Kubernetes cluster is running OLM already you can use the `operator-sdk` command to find out.

~~~sh
operator-sdk olm status
~~~

If your cluster is not running olm and you want to deploy it, you can run the following command:

~~~sh
operator-sdk olm install
~~~

Once we know OLM is present in our cluster we can continue and start the creation of our Operator Bundle.

## Operator Bundle

An Operator Bundle consists of different manifests (CSVs and CRDs) and some metadata that defines the Operator at a specific version.

You can read more about Bundles [here](https://github.com/operator-framework/operator-registry/blob/v1.12.6/docs/design/operator-bundle.md).

## Requirements

At the moment of this writing the following versions were used:

* golang-1.19.5
* Operator Framework SDK v1.26.1
* Kubernetes 1.24
* opm v1.26.3

### Creating the Operator Bundle

We need to change to the Reverse Words Operator folder and run the `make bundle` command. We will be asked for some information.

~~~sh
cd ~/operators-projects/reverse-words-operator/

QUAY_USERNAME=<username>
make bundle VERSION=0.0.1 CHANNELS=alpha DEFAULT_CHANNEL=alpha IMG=quay.io/$QUAY_USERNAME/reversewords-operator:v0.0.1
~~~

> **NOTE**: Example output

~~~
Display name for the operator (required): 
> Reverse Words Operator

Description for the operator (required): 
> Deploys and Manages instances of the Reverse Words Application

Provider's name for the operator (required): 
> Linuxera

Any relevant URL for the provider name (optional): 
> linuxera.org

Comma-separated list of keywords for your operator (required): 
> reverse,reversewords,linuxera

Comma-separated list of maintainers and their emails (e.g. 'name1:email1, name2:email2') (required): 
> mario@linuxera.org

<omitted_output>
~~~

Above command has generated some files:

~~~
bundle
├── manifests
│   ├── apps.linuxera.org_reversewordsapps.yaml
│   ├── reverse-words-operator.clusterserviceversion.yaml
│   ├── reverse-words-operator-controller-manager-metrics-service_v1_service.yaml
│   ├── reverse-words-operator-manager-config_v1_configmap.yaml
│   └── reverse-words-operator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
├── metadata
│   └── annotations.yaml
└── tests
    └── scorecard
        └── config.yaml
~~~

We need to tweak the ClusterServiceVersion a bit:

1. Configure proper `installModes`
2. Add WATCH_NAMESPACE env var to the operator deployment
3. Add an Icon to our Operator

You can download the modified CSV here:

~~~sh
curl -Ls https://linuxera.org/integrating-operators-olm/reverse-words-operator.clusterserviceversion_v0.0.1.yaml -o ~/operators-projects/reverse-words-operator/bundle/manifests/reverse-words-operator.clusterserviceversion.yaml
sed -i "s/QUAY_USER/$QUAY_USERNAME/g" ~/operators-projects/reverse-words-operator/bundle/manifests/reverse-words-operator.clusterserviceversion.yaml
~~~

Now that we have the Operator Bundle ready we can build it and push it to [Quay](https://quay.io). After that we will build the catalog image and once the catalog image is ready, we will use it to deploy our operator.

> **NOTE**: If you use podman instead of docker you can edit the Makefile and change docker commands by podman commands

1. Build and Push the bundle

    ~~~sh
    make bundle-build bundle-push BUNDLE_IMG=quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.1
    ~~~
2. Validate the bundle

    ~~~sh
    operator-sdk bundle validate quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.1 -b podman
    ~~~
3. Create the Catalog Image

    {{<tip>}}
With this new file-based approach, it's highly recommended to keep control of your catalog in Git. The operator framework team created an example repo you can fork [here](https://github.com/operator-framework/cool-catalog).
    {{</tip>}}

    ~~~sh
    # Download opm tool
    sudo curl -sL https://github.com/operator-framework/operator-registry/releases/download/v1.26.3/linux-amd64-opm -o /usr/local/bin/opm && sudo chmod +x /usr/local/bin/opm
    # Create the catalog image
    mkdir reversewords-catalog
    opm generate dockerfile reversewords-catalog
    opm init reverse-words-operator --default-channel=alpha --output yaml > reversewords-catalog/operator.yaml
    # Add bundle
    opm render quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.1 --output yaml >> reversewords-catalog/operator.yaml
    # Initialize the alpha channel
    cat << EOF >> reversewords-catalog/operator.yaml
    ---
    schema: olm.channel
    package: reverse-words-operator
    name: alpha
    entries:
      - name: reverse-words-operator.v0.0.1
    EOF
    # Validate the catalog
    opm validate reversewords-catalog && echo "OK"
    ~~~
4. Building and pushing the catalog image

    ~~~sh
    podman build . -f reversewords-catalog.Dockerfile -t quay.io/$QUAY_USERNAME/reversewords-catalog:latest
    podman push quay.io/$QUAY_USERNAME/reversewords-catalog:latest
    ~~~

## Deploy the Operator using OLM

At this point we have our bundle and catalog images ready, we just need to create the required `CatalogSource` into the cluster so we get access to our Operator bundle.

~~~sh
OLM_NAMESPACE=$(kubectl get pods -A | grep catalog-operator | awk '{print $1}')
cat <<EOF | kubectl create -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: reversewords-catalog
  namespace: $OLM_NAMESPACE
spec:
  sourceType: grpc
  displayName: "ReverseWords Catalog"
  publisher: Linuxera
  image: quay.io/$QUAY_USERNAME/reversewords-catalog:latest
  updateStrategy:
    registryPoll:
      interval: 1m0s
EOF
~~~

A pod will be created on the OLM namespace:

~~~sh
kubectl -n $OLM_NAMESPACE get pod -l olm.catalogSource=reversewords-catalog

NAME                         READY   STATUS    RESTARTS   AGE
reversewords-catalog-d8qbw   1/1     Running   0          12s
~~~

OLM will read the CSVs from our Operator Bundle and will load the Package Manifest into the cluster:

~~~sh
kubectl get packagemanifest -l catalog=reversewords-catalog
NAME                     CATALOG                AGE
reverse-words-operator   ReverseWords Catalog   30s
~~~

At this point we can create a `Subscription` to our operator:

1. Create a new namespace

    ~~~sh
    NAMESPACE=test-operator-subscription
    kubectl create ns $NAMESPACE
    ~~~
2. Create the subscription in the namespace we just created

    ~~~sh
    cat <<EOF | kubectl -n $NAMESPACE create -f - 
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: reversewords-subscription
    spec:
      channel: alpha
      name: reverse-words-operator
      installPlanApproval: Automatic
      source: reversewords-catalog
      sourceNamespace: $OLM_NAMESPACE
    ---
    apiVersion: operators.coreos.com/v1
    kind: OperatorGroup
    metadata:
      name: reverse-words-operatorgroup
    spec:
      targetNamespaces:
      - $NAMESPACE
    EOF
    ~~~
3. The operator will be deployed in the namespace

    ~~~sh
    kubectl -n $NAMESPACE get pods

    NAME                                                         READY   STATUS             RESTARTS   AGE
    reverse-words-operator-controller-manager-844d897db4-hsnmw   2/2     Running   0          12s
    ~~~

## Publish an upgrade for our Operator

We have seen how to create a bundle for our operator, now we are going to see how we can add new versions to the bundle and link them so we can publish updates for the operators.

In the previous steps we create the version `v0.0.1` for our Operator, we are going to create and publish a `v0.0.2` version:

First we create a new CSV version in the same channel we used for v0.0.1:

~~~sh
make bundle VERSION=0.0.2 CHANNELS=alpha DEFAULT_CHANNEL=alpha IMG=quay.io/$QUAY_USERNAME/reversewords-operator:v0.0.2
~~~

We need to tweak the ClusterServiceVersion a bit:

1. Configure proper `installModes`
2. Add WATCH_NAMESPACE env var to the operator deployment
3. Add an Icon to our Operator

You can download the modified CSV here:

~~~sh
curl -Ls https://linuxera.org/integrating-operators-olm/reverse-words-operator.clusterserviceversion_v0.0.2.yaml -o ~/operators-projects/reverse-words-operator/bundle/manifests/reverse-words-operator.clusterserviceversion.yaml
sed -i "s/QUAY_USER/$QUAY_USERNAME/g" ~/operators-projects/reverse-words-operator/bundle/manifests/reverse-words-operator.clusterserviceversion.yaml
~~~

Now that we have the new Operator Bundle ready we can build it and push it to [Quay](https://quay.io). After that we will update and build the catalog image.

1. Build and push the new bundle

    ~~~sh
    make bundle-build bundle-push BUNDLE_IMG=quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.2
    ~~~
2. Validate the new bundle

    ~~~sh
    operator-sdk bundle validate quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.2 -b podman
    ~~~
3. Update the Catalog Image

    ~~~sh
    # Add bundle
    opm render quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.2 --output yaml >> reversewords-catalog/operator.yaml
    # Update the alpha channel: Edit file reversewords-catalog/operator.yaml and change
    entries:
      - name: reverse-words-operator.v0.0.1
      - name: reverse-words-operator.v0.0.2
        replaces: reverse-words-operator.v0.0.1
    # Validate the catalog
    opm validate reversewords-catalog && echo "OK"
    ~~~

4. Build and push the catalog image

    ~~~sh
    podman build . -f reversewords-catalog.Dockerfile -t quay.io/$QUAY_USERNAME/reversewords-catalog:latest
    podman push quay.io/$QUAY_USERNAME/reversewords-catalog:latest
    ~~~

OLM is continuously checking for new bundles in the Catalog Images based on the `updateStrategy` we configured in our `CatalogSource`. After a few moments, the new CSV `v0.0.2` should be available.

## Updating to a new Operator version

Depending on the `installPlanApproval` you selected when you created the subscription the operator will be updated automatically when a new version is published or you may need to approve the `installPlan` so the operator gets updated.

# Sources

* [OLM Generation Docs](https://github.com/operator-framework/operator-sdk/blob/master/website/content/en/docs/olm-integration/generation.md)
* [Operator Registry Docs](https://github.com/operator-framework/operator-registry/blob/master/README.md)
* [Operator SDK OLM Integration Bundle Quickstart](https://sdk.operatorframework.io/docs/olm-integration/quickstart-bundle/)
* [Operator SDK Generating Manifests and Metadata](https://sdk.operatorframework.io/docs/olm-integration/generation/)
