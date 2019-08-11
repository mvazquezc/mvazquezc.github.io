---
layout: post
title:  "Federating MongoDB on OKD with Federation V2"
author: "Mario"
categories: [ okd, origin, openshift, containers, kubernetes, federation, federationv2 ]
featured: false
image: assets/images/2019-01-25-federating-mongodb-with-federationv2.jpg
image-author: "Erwan Hesry"
image-author-link: "https://unsplash.com/photos/RJjY5Hpnifk?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText"
image-source: "Unsplash"
image-source-link: "https://unsplash.com/search/photos/container?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText"
permalink: /federating-mongodb-on-okd-with-federation-v2/
hidden: false
---

# Scenario

We will start with three independent **OKD 3.11** clusters. Once deployed, we will federate them using [Federation V2](https://github.com/kubernetes-sigs/federation-v2).

With the clusters federated we will deploy a **MongoDB ReplicaSet** across the clusters (One replica per cluster).

## Deploying three AIO OKD 3.11 Clusters

I've created three CentOS 7 instances, on top of those instances OKD 3.11 AIO was deployed using [this tool](https://github.com/mvazquezc/installcentos).

I do own a domain, so the required DNS registries were created there. If you don't have/want to use a custom domain, you can use nip.io/xip.io for the matter.

My setup looks like:

* Cluster1
  * Hardware: 2vCPU and 6GB RAM
  * DNS: cluster1.linuxlabs.org
  * Wildcard: \*.cluster1.linuxlabs.org
* Cluster2
  * Hardware: 2vCPU and 6GB RAM
  * DNS: cluster2.linuxlabs.org
  * Wildcard: \*.cluster2.linuxlabs.org
* Cluster3
  * Hardware: 2vCPU and 6GB RAM
  * DNS: cluster3.linuxlabs.org
  * Wildcard: \*.cluster3.linuxlabs.org

## Federating AIO OKD 3.11 Clusters

Once OKD is installed, login on each cluster and rename the context so you end up with something like this:

```
cluster1  console-cluster1-linuxlabs-org:8443  admin/console-cluster1-linuxlabs-org:8443  default
cluster2  console-cluster2-linuxlabs-org:8443  admin/console-cluster2-linuxlabs-org:8443  default
cluster3  console-cluster3-linuxlabs-org:8443  admin/console-cluster3-linuxlabs-org:8443  default
```

Now it's time to federate those clusters using Federation V2, the most updated documentation can be found in the [userguide](https://github.com/kubernetes-sigs/federation-v2/blob/master/docs/userguide.md). 

Below the steps followed at the time of writing (subject to change in the future).

```sh
# Create Contexts
oc config delete-context cluster1
oc config delete-context cluster2
oc config delete-context cluster3
oc login https://console.cluster1.linuxlabs.org:8443/
oc config rename-context $(oc config current-context) cluster1
oc login https://console.cluster2.linuxlabs.org:8443/
oc config rename-context $(oc config current-context) cluster2
oc login https://console.cluster3.linuxlabs.org:8443/
oc config rename-context $(oc config current-context) cluster3
# Deploy Federation V2
git clone https://github.com/kubernetes-sigs/federation-v2
cd federation-v2
oc config use cluster1
oc create ns federation-system
oc create ns kube-multicluster-public
oc create clusterrolebinding federation-admin --clusterrole=cluster-admin --serviceaccount="federation-system:default"
oc -n federation-system apply --validate=false -f hack/install-latest.yaml
oc apply --validate=false -f vendor/k8s.io/cluster-registry/cluster-registry-crd.yaml
for filename in ./config/federatedirectives/*.yaml; do kubefed2 federate enable -f "${filename}" --federation-namespace=federation-system; done
cd .. && rm -rf federation-v2/
# Join Clusters to Federation Control Plane
kubefed2 join cluster1 --host-cluster-context cluster1 --add-to-registry --v=2 --federation-namespace=federation-system
kubefed2 join cluster2 --host-cluster-context cluster1 --add-to-registry --v=2 --federation-namespace=federation-system
kubefed2 join cluster3 --host-cluster-context cluster1 --add-to-registry --v=2 --federation-namespace=federation-system
```

## Deploying a Federated MongoDB ReplicaSet

**Resources**

* [00-mongo-federated-namespace.yaml](https://linuxera.org/assets/post_resources/2019-01-25-federating-mongodb-with-federationv2/00-mongo-federated-namespace.yaml)
* [01-mongo-federated-secret.yaml](https://linuxera.org/assets/post_resources/2019-01-25-federating-mongodb-with-federationv2/01-mongo-federated-secret.yaml)
* [02-mongo-federated-service.yaml](https://linuxera.org/assets/post_resources/2019-01-25-federating-mongodb-with-federationv2/02-mongo-federated-service.yaml)
* [03-mongo-federated-pvc.yaml](https://linuxera.org/assets/post_resources/2019-01-25-federating-mongodb-with-federationv2/03-mongo-federated-pvc.yaml)
* [04-mongo-federated-deployment-rs.yaml](https://linuxera.org/assets/post_resources/2019-01-25-federating-mongodb-with-federationv2/04-mongo-federated-deployment-rs.yaml)

1. First, a Namespace and a FederatedNamespacePlacement is created into the cluster running the Federation Control Plane, Cluster1 in this case
    
    ```sh
    oc --context=cluster1 create -f 00-mongo-federated-namespace.yaml
    ```
  1. At this point we have the "federated-mongo" Namespace across three different clusters (Cluster1, Cluster3 and Cluster2)
      
      ```sh
      for cluster in cluster1 cluster2 cluster3;do echo "**Cluster ${cluster}**";oc --context=$cluster get namespaces | grep federated-mongo;done
      **Cluster cluster1**
      federated-mongo            Active    11s
      **Cluster cluster2**
      federated-mongo            Active    15s
      **Cluster cluster3**
      federated-mongo            Active    18s
      ```
2. MongoDB communications will be secured using SSL, so we need to generate valid certificates for our route names
  1. Currently, we need to know the route hostnames beforehand and generate the certs manually, there is room for improvement on this area
  2. You should generate your own certs and modify the federated secret definition
      
      ```sh
      cat 01-mongo-federated-secret.yaml | grep mongodb.pem | awk -F ": " '{print $2}' | base64 -d | openssl x509 -text | grep DNS
      DNS:localhost, DNS:localhost.localdomain, DNS:mongo.apps.cluster1.linuxlabs.org, DNS:mongo.apps.cluster3.linuxlabs.org, DNS:mongo.apps.cluster2.linuxlabs.org, DNS:mongo, DNS:mongo.federated-mongo, DNS:mongo.federated-mongo.svc.cluster.local, IP Address:127.0.0.1

      oc --context=cluster1 create -f 01-mongo-federated-secret.yaml
      ```
  3. A federatedSecret has been created across the federated clusters, the secrets include the certificates and user/password details
      
      ```sh
      for cluster in cluster1 cluster2 cluster3;do echo "**Cluster ${cluster}**";oc --context=$cluster -n federated-mongo get secrets | grep mongodb;done
      **Cluster cluster1**
      mongodb-secret             Opaque                                5         25s
      mongodb-ssl                Opaque                                2         25s
      **Cluster cluster2**
      mongodb-secret             Opaque                                5         27s
      mongodb-ssl                Opaque                                2         27s
      **Cluster cluster3**
      mongodb-secret             Opaque                                5         29s
      mongodb-ssl                Opaque                                2         29s
      ```
3. We need a service on each cluster, so we are going to create a federatedservice for that purpouse 
    
    ```sh
    oc --context=cluster1 create -f 02-mongo-federated-service.yaml
    for cluster in cluster1 cluster2 cluster3;do echo "**Cluster ${cluster}**";oc --context=$cluster -n federated-mongo get services;done
    **Cluster cluster1**
    NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
    mongo     ClusterIP   172.30.202.16   <none>        27017/TCP   1s
    **Cluster cluster2**
    NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
    mongo     ClusterIP   172.30.136.75   <none>        27017/TCP   3s
    **Cluster cluster3**
    NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
    mongo     ClusterIP   172.30.41.162   <none>        27017/TCP   4s
    ```
4. Our deployment needs a volume in order to store the MongoDB data, so let's federate the PVC Object and then create the Federated resource definition, so we will get a PVC on each cluster

    ```sh
    kubefed2 federate enable PersistentVolumeClaim --host-cluster-context cluster1
    oc --context=cluster1 create -f 03-mongo-federated-pvc.yaml
    for cluster in cluster1 cluster2 cluster3;do echo "**Cluster ${cluster}**";oc --context=$cluster -n federated-mongo get pvc;done
    **Cluster cluster1**
    NAME      STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    mongo     Bound     vol107    500Gi      RWO,RWX                       1s
    **Cluster cluster2**
    NAME      STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    mongo     Bound     vol68     500Gi      RWO,RWX                       3s
    **Cluster cluster3**
    NAME      STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    mongo     Bound     vol47     500Gi      RWO,RWX                       4s
    ```
5. Now, we are ready to deploy the MongoDB Replicas, for this demo we will be federating a deployment with one replica. So we will have three MongoDB pods, one pod on each cluster.
    
    ```sh
    oc --context=cluster1 create -f 04-mongo-federated-deployment-rs.yaml
    for cluster in cluster1 cluster2 cluster3;do echo "**Cluster ${cluster}**";oc --context=$cluster -n federated-mongo get pods;done
    **Cluster cluster1**
    NAME                     READY     STATUS    RESTARTS   AGE
    mongo-6cfb7cd4df-9hpzw   1/1       Running   0          31s
    **Cluster cluster2**
    NAME                     READY     STATUS    RESTARTS   AGE
    mongo-6cfb7cd4df-6b7kn   1/1       Running   0          31s
    **Cluster cluster3**
    NAME                     READY     STATUS    RESTARTS   AGE
    mongo-6cfb7cd4df-nf2sc   1/1       Running   0          33s
    ```
6. Finally, we need to create the routes in order to get external traffic to our pods, these routes will be passtrhough as we need mongo to handle the certs and the connection to remain raw TCP rather than HTTPS
    
    NOTE: Check route hostnames are not already in use!

    ```sh
    oc --context=cluster1 -n federated-mongo create route passthrough mongo --service=mongo --port=27017 --hostname=mongo.apps.cluster1.linuxlabs.org
    oc --context=cluster2 -n federated-mongo create route passthrough mongo --service=mongo --port=27017 --hostname=mongo.apps.cluster2.linuxlabs.org
    oc --context=cluster3 -n federated-mongo create route passthrough mongo --service=mongo --port=27017 --hostname=mongo.apps.cluster3.linuxlabs.org
    ```
7. Next we are going to configure the MongoDB ReplicaSet, this procedure has been automated and the only thing you need to do is label the primary pod, in this case Cluster1
    
    ```sh
    MONGO_POD=$(oc --context=cluster1 -n federated-mongo get pod --selector="name=mongo" --output=jsonpath='{.items..metadata.name}')
    oc --context=cluster1 -n federated-mongo label pod $MONGO_POD replicaset=primary
    ```

8. After some seconds, the MongoDB ReplicaSet will be ready, we can connect to one replica an check the status

    ```sh
    MONGO_POD=$(oc --context=cluster1 -n federated-mongo get pod --selector="name=mongo" --output=jsonpath='{.items..metadata.name}')
    oc --context=cluster1 -n federated-mongo rsh $MONGO_POD
    mongo --host localhost admin --tls --tlsCAFile /opt/mongo-ssl/ca.pem --eval "var adm_pwd = '$MONGODB_ADMIN_PASSWORD'" --shell
    db.auth('admin',adm_pwd)
    rs.status()
    ```

# Final notes

Now we have a MongoDB ReplicaSet distributed across diferent clusters ready to be used by our applications. As you have seen, the deployment has been orchestrated from the Federated Host Cluster (Cluster1) in an easy and intuitive way.

If you want to learn more about Federation V2, take a look at this [Katacoda Scenario](https://learn.openshift.com/introduction/federated-clusters/).

Federation V2 is under heavy active development, bugs and changes are expected as it's still an alpha.

Don't forget to share your thoughts in the comments below.
