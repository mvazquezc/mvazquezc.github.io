---
layout: post
title:  "Integrating our Operators with OLM"
author: "Mario"
categories: [ okd, origin, containers, kubernetes, operators, controllers, operator framework, operator sdk, operator lifecycle manager, olm ]
featured: false
image: assets/images/2020-09-16-operator-sdk-olm-integration.jpg
image-author: "Ammiel Jr"
image-author-link: "https://unsplash.com/@helloitsammiel"
image-source: "Unsplash"
image-source-link: "https://unsplash.com/photos/qX2ENCIxquA"
permalink: /integrating-operators-olm/
hidden: false
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

cd config/manager && /home/mario/go/bin/kustomize edit set image controller=quay.io/mavazque/reversewords-operator:v0.0.1
/home/mario/go/bin/kustomize build config/manifests | operator-sdk generate bundle -q --overwrite --version 0.0.1 --channels=alpha --default-channel=alpha
INFO[0000] Building annotations.yaml                    
INFO[0000] Writing annotations.yaml in /home/mario/operators-projects/reverse-words-operator/bundle/metadata 
INFO[0000] Building Dockerfile                          
INFO[0000] Writing bundle.Dockerfile in /home/mario/operators-projects/reverse-words-operator 
operator-sdk bundle validate ./bundle
INFO[0000] Found annotations file                        bundle-dir=bundle container-tool=docker
INFO[0000] Could not find optional dependencies file     bundle-dir=bundle container-tool=docker
INFO[0000] All validation tests have completed successfully 
~~~

Above command has generated some files:

~~~
bundle
????????? manifests
??????? ????????? apps.linuxera.org_reversewordsapps.yaml
??????? ????????? reverse-words-operator.clusterserviceversion.yaml
??????? ????????? reverse-words-operator-metrics-reader_rbac.authorization.k8s.io_v1beta1_clusterrole.yaml
????????? metadata
??????? ????????? annotations.yaml
????????? tests
    ????????? scorecard
        ????????? config.yaml
~~~

We need to tweak the ClusterServiceVersion a bit:

1. Configure proper `installModes`
2. Add WATCH_NAMESPACE env var to the operator deployment
3. Add an Icon to our Operator

You can download the modified CSV here:

~~~sh
curl -Ls https://linuxera.org/assets/post_resources/2020-09-16-operator-sdk-olm-integration/reverse-words-operator.clusterserviceversion_v0.0.1.yaml -o ~/operators-projects/reverse-words-operator/bundle/manifests/reverse-words-operator.clusterserviceversion.yaml
sed -i "s/QUAY_USER/$QUAY_USERNAME/g" ~/operators-projects/reverse-words-operator/bundle/manifests/reverse-words-operator.clusterserviceversion.yaml
~~~

Now that we have the Operator Bundle ready we can build it and push it to [Quay](https://quay.io). After that we will build the index image and once the index image is ready, we will use it to deploy our operator.

> **NOTE**: I'll be using podman, you can use docker as well

1. Build the bundle

    ~~~sh
    podman build -f bundle.Dockerfile -t quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.1
    ~~~
2. Push and validate the bundle

    ~~~sh
    podman push quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.1
    operator-sdk bundle validate quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.1 -b podman
    ~~~
3. Create the [Index Image](https://github.com/operator-framework/operator-registry#building-an-index-of-operators-using-opm)

    ~~~sh
    # Download opm tool
    sudo curl -sL https://github.com/operator-framework/operator-registry/releases/download/v1.13.8/linux-amd64-opm -o /usr/local/bin/opm && chmod +x /usr/local/bin/opm
    # Create the index image
    opm index add -c podman --bundles quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.1 --tag quay.io/$QUAY_USERNAME/reversewords-index:v0.0.1
    # Push the index image
    podman push quay.io/$QUAY_USERNAME/reversewords-index:v0.0.1
    ~~~

## Deploy the Operator using OLM

At this point we have our bundle and index image ready, we just need to create the required `CatalogSource` into the cluster so we get access to our Operator bundle.

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
  image: quay.io/$QUAY_USERNAME/reversewords-index:v0.0.1
EOF
~~~

A pod will be created on the OLM namespace:

~~~sh
kubectl -n $OLM_NAMESPACE get pod -l olm.catalogSource=reversewords-catalog

NAME                         READY   STATUS    RESTARTS   AGE
reversewords-catalog-jdn78   1/1     Running   0          3m11s
~~~

OLM will read the CSVs from our Operator Bundle and will load the Package Manifest into the cluster:

~~~sh
kubectl get packagemanifest -l catalog=reversewords-catalog
NAME                     CATALOG   AGE
reverse-words-operator             4m9s
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
    reverse-words-operator-controller-manager-7c57649d7f-x88w5   2/2     Running            0          5m5s
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
curl -Ls https://linuxera.org/assets/post_resources/2020-09-16-operator-sdk-olm-integration/reverse-words-operator.clusterserviceversion_v0.0.2.yaml -o ~/operators-projects/reverse-words-operator/bundle/manifests/reverse-words-operator.clusterserviceversion.yaml
sed -i "s/QUAY_USER/$QUAY_USERNAME/g" ~/operators-projects/reverse-words-operator/bundle/manifests/reverse-words-operator.clusterserviceversion.yaml
~~~

Now that we have the new Operator Bundle ready we can build it and push it to [Quay](https://quay.io). After that we will update and build the index image.

1. Build the new bundle

    ~~~sh
    podman build -f bundle.Dockerfile -t quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.2
    ~~~
2. Push and validate the new bundle

    ~~~sh
    podman push quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.2
    operator-sdk bundle validate quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.2 -b podman
    ~~~
3. Update the Index Image

    ~~~sh
    # Create the index image
    opm index add -c podman --bundles quay.io/$QUAY_USERNAME/reversewords-operator-bundle:v0.0.2 --from-index quay.io/$QUAY_USERNAME/reversewords-index:v0.0.1 --tag quay.io/$QUAY_USERNAME/reversewords-index:v0.0.2
    # Push the index image
    podman push quay.io/$QUAY_USERNAME/reversewords-index:v0.0.2
    ~~~

With the index image updated, we can now update the `CatalogSource` pointing to the new index image:

~~~sh
PATCH="{\"spec\":{\"image\":\"quay.io/$QUAY_USERNAME/reversewords-index:v0.0.2\"}}"
kubectl -n $OLM_NAMESPACE patch catalogsource reversewords-catalog -p $PATCH --type merge
~~~

The catalog pod will be recreated with the new index image and the package manifest will be updated to include the CSV `v0.0.2`.

## Updating to a new Operator version

Depending on the `installPlanApproval` you selected when you created the subscription the operator will be updated automatically when a new version is published or you may need to approve the `installPlan` so the operator gets updated.

# Sources

* [OLM Generation Docs](https://github.com/operator-framework/operator-sdk/blob/master/website/content/en/docs/olm-integration/generation.md)
* [Operator Registry Docs](https://github.com/operator-framework/operator-registry/blob/master/README.md)
* [Operator SDK OLM Integration Bundle Quickstart](https://sdk.operatorframework.io/docs/olm-integration/quickstart-bundle/)
* [Operator SDK Generating Manifests and Metadata](https://sdk.operatorframework.io/docs/olm-integration/generation/)
