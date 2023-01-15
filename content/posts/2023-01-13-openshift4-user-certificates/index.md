---
title:  "OpenShift 4 User Certificates"
author: "Mario"
tags: [ "openshift", "opc", "security", "client-certificates", "users" ]
url: "/user-certificates-in-openshift4/"
draft: false
date: 2023-01-13
#lastmod: 2023-01-13
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# User Certificates in OpenShift 4

{{<attention>}}
The information described in this blog post may not be a supported configuration for OpenShift 4. Please, refer to the [official docs](https://docs.openshift.com) for supported documentation.
{{</attention>}}

In this blog we will see how we can create **OpenShift Users** using client certificates and how to configure the API Server, so we can create client certificates using **custom CAs**. The information described in this blog was last tested with OpenShift 4.11.

## OpenShift Authentication

As you may know, OpenShift 4 supports different authentication providers. By default, once your cluster is installed you will get a `kubeadmin` user to access the UI and a `kubeconfig` file configured with a client cert to access the API Server as cluster-admin.

On top of that you get an OAuth Server that can be configured for adding different authentication sources like GitHub, LDAP, etc. This post will focus on creating new client certificates to access the API Server via `kubeconfig` files.

The scenario will be as follows:

- A client certificate for the user `luffy` will be issued using a self-signed CA named `customer-signer-custom`.
- A client certificate for the user `zoro` will be issued using a self-signed CA named `customer-signer-custom-2`.
- A client certificate for the user `nami` will be issued using the internal OpenShift CA via a `CertificateSigningRequest`.

### Creating the client certificate for Nami using the internal OpenShift CA

Before we create this first user, we need to understand how usernames and groups are assigned in Kubernetes when using client certificates. Groups for the user will be configured in the `Organization` field while username will be configured in the `Common Name` field.

Let's get startes by creating a CSR for the client certificate using the `openssl` client:

{{<tip>}}
We are using group "system:admin" and username "nami" for this client certificate.
{{</tip>}}

~~~sh
openssl req -nodes -newkey rsa:4096 -keyout /tmp/nami.key -subj "/O=system:admin/CN=nami" -out /tmp/nami.csr
~~~

Now that we have the csr, let's submit the CSR to OpenShift in order to sign it with the internal CA later:

~~~sh
cat << EOF | oc create -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: nami-access
spec:
  signerName: kubernetes.io/kube-apiserver-client
  groups:
  - system:authenticated
  request: $(cat /tmp/nami.csr | base64 -w0)
  usages:
  - client auth
EOF
~~~

Next, time to approve the certificate request:

~~~sh
oc adm certificate approve nami-access
~~~

The certificate has been approved, we can get it now from the cluster:

~~~sh
oc get csr nami-access -o jsonpath='{.status.certificate}' | base64 -d > /tmp/nami.crt
~~~

At this point we're all set, next step will be creating a `kubeconfig` file for `nami` to log into our cluster, we will leave that for later. For now, let's continue creating the client certificates for `luffy` and `zoro`.

### Creating the client certificate for Luffy using the customer-signer-custom self-signed CA

We need a self-signed CA, although this could be an existing CA that already exists in your environment, for this post we will create one using the `openssl` client:

~~~sh
openssl genrsa -out /tmp/customer-ca.key 4096
openssl req -x509 -new -nodes -key /tmp/customer-ca.key -sha256 -days 9999 -out /tmp/customer-ca.crt -subj "/OU=openshift/CN=customer-signer-custom"
~~~

Now that we have the CA, let's create the `luffy` client certificate request:

{{<tip>}}
We are using group "system:admin" and username "luffy" for this client certificate.
{{</tip>}}

~~~sh
openssl req -nodes -newkey rsa:4096 -keyout /tmp/luffy.key -subj "/O=system:admin/CN=luffy" -out /tmp/luffy.csr
~~~

We can now sign the csr:

~~~sh
openssl x509 -extfile <(printf "extendedKeyUsage = clientAuth") -req -in /tmp/luffy.csr -CA /tmp/customer-ca.crt -CAkey /tmp/customer-ca.key -CAcreateserial -out /tmp/luffy.crt -days 9999 -sha256
~~~

And we're done here, next step would be the `kubeconfig` creation. Let's continue with `zoro's` client certificate before we start creating the `kubeconfig` files.

### Creating the client certificate for Zoro using the customer-signer-custom-2 self-signed CA

{{<tip>}}
We are using group "system:admin" and username "zoro" for this client certificate.
{{</tip>}}

~~~sh
# Create self-signed CA
openssl genrsa -out /tmp/customer-ca-2.key 4096
openssl req -x509 -new -nodes -key /tmp/customer-ca-2.key -sha256 -days 9999 -out /tmp/customer-ca-2.crt -subj "/OU=openshift/CN=customer-signer-custom-2"
# Create CSR for zoro's client cert
openssl req -nodes -newkey rsa:4096 -keyout /tmp/zoro.key -subj "/O=system:admin/CN=zoro" -out /tmp/zoro.csr
# Sign CSR
openssl x509 -extfile <(printf "extendedKeyUsage = clientAuth") -req -in /tmp/zoro.csr -CA /tmp/customer-ca-2.crt -CAkey /tmp/customer-ca-2.key -CAcreateserial -out /tmp/zoro.crt -days 9999 -sha256
~~~

### Creating the kubeconfig files for Nami, Luffy and Zoro

Before we start creating the kubeconfig files we need to get the public cert of our API server, this will be used by the kubectl/oc clients in order to trust the API Server certificate.

~~~sh
export OPENSHIFT_API_SERVER_ENDPOINT=api.cluster.example.com:6443
openssl s_client -showcerts -connect ${OPENSHIFT_API_SERVER_ENDPOINT} </dev/null 2>/dev/null|openssl x509 -outform PEM > /tmp/ocp-apiserver-cert.crt
~~~

Let's start with `Nami's` kubeconfig:

~~~sh
oc --kubeconfig /tmp/nami config set-credentials nami --client-certificate=/tmp/nami.crt --client-key=/tmp/nami.key --embed-certs=true
oc --kubeconfig /tmp/nami config set-cluster openshift-cluster-dev --certificate-authority=/tmp/ocp-apiserver-cert.crt --embed-certs=true --server=https://${OPENSHIFT_API_SERVER_ENDPOINT}
oc --kubeconfig /tmp/nami config set-context openshift-dev --cluster=openshift-cluster-dev --namespace=default --user=nami
oc --kubeconfig /tmp/nami config use-context openshift-dev
~~~

Now let's do `Zoro's`:

~~~sh
oc --kubeconfig /tmp/zoro config set-credentials zoro --client-certificate=/tmp/zoro.crt --client-key=/tmp/zoro.key --embed-certs=true
oc --kubeconfig /tmp/zoro config set-cluster openshift-cluster-dev --certificate-authority=/tmp/ocp-apiserver-cert.crt --embed-certs=true --server=https://${OPENSHIFT_API_SERVER_ENDPOINT}
oc --kubeconfig /tmp/zoro config set-context openshift-dev --cluster=openshift-cluster-dev --namespace=default --user=zoro
oc --kubeconfig /tmp/zoro config use-context openshift-dev
~~~

And finally `Luffy's`:

~~~sh
oc --kubeconfig /tmp/luffy config set-credentials luffy --client-certificate=/tmp/luffy.crt --client-key=/tmp/luffy.key --embed-certs=true
oc --kubeconfig /tmp/luffy config set-cluster openshift-cluster-dev --certificate-authority=/tmp/ocp-apiserver-cert.crt --embed-certs=true --server=https://${OPENSHIFT_API_SERVER_ENDPOINT}
oc --kubeconfig /tmp/luffy config set-context openshift-dev --cluster=openshift-cluster-dev --namespace=default --user=luffy
oc --kubeconfig /tmp/luffy config use-context openshift-dev
~~~

### Accessing our cluster with the new kubeconfig files:

~~~sh
oc --kubeconfig /tmp/luffy whoami
error: You must be logged in to the server (Unauthorized)

oc --kubeconfig /tmp/zoro whoami
error: You must be logged in to the server (Unauthorized)

oc --kubeconfig /tmp/nami whoami
nami
~~~

As you can see from the above output only `Nami` can access the cluster, but why? - Well, the API Server doesn't trust the self-signed CAs we created. We need to configure it to trust them, let's do it:

{{<tip>}}
Since we have two self-signed CAs and the APIServer only accepts 1 ConfigMap, we need to create a bundle with all the CAs we want to trust when using client certificates authentication. This doesn't include the internal OpenShift CA, which is always trusted.
{{</tip>}}

~~~sh
cat /tmp/customer-ca.crt /tmp/customer-ca-2.crt > /tmp/customer-custom-cas.crt

oc create configmap customer-cas-custom -n openshift-config --from-file=ca-bundle.crt=/tmp/customer-custom-cas.crt
~~~

Now that the `ConfigMap` is ready, let's tell the APIServer to use it:

~~~sh
oc patch apiserver cluster --type=merge -p '{"spec": {"clientCA": {"name": "customer-cas-custom"}}}'
~~~

If we try the kubeconfig files again, this is the result:

~~~sh
oc --kubeconfig /tmp/luffy whoami
luffy

oc --kubeconfig /tmp/zoro whoami
zoro

oc --kubeconfig /tmp/nami whoami
nami
~~~

Now all three work, but if we try to do other stuff like listing pods, nodes, etc. we will see that we don't have access to that. That's expected since in a default OCP installation we don't have RBAC rules for the `system:admin` group:

~~~sh
oc --kubeconfig /tmp/luffy get nodes
Error from server (Forbidden): nodes is forbidden: User "luffy" cannot list resource "nodes" in API group "" at the cluster scope

oc --kubeconfig /tmp/zoro -n default get pods
Error from server (Forbidden): pods is forbidden: User "zoro" cannot list resource "pods" in API group "" in the namespace "default"

oc --kubeconfig /tmp/nami -n default get deployments
Error from server (Forbidden): deployments.apps is forbidden: User "nami" cannot list resource "deployments" in API group "apps" in the namespace "default"
~~~

Let's configure users in the `system:admin` group as cluster admins:

~~~sh
oc adm policy add-cluster-role-to-group cluster-admin system:admin
~~~

If we try again:

~~~sh
oc --kubeconfig /tmp/luffy get nodes
NAME                 STATUS   ROLES           AGE   VERSION
openshift-master-0   Ready    master,worker   99d   v1.24.0+b62823b
openshift-master-1   Ready    master,worker   99d   v1.24.0+b62823b
openshift-master-2   Ready    master,worker   99d   v1.24.0+b62823b

oc --kubeconfig /tmp/zoro -n default get pods
NAME                    READY   STATUS    RESTARTS   AGE
test-64ccd87d6c-98j45   1/1     Running   1          4d4h

oc --kubeconfig /tmp/nami -n default get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
test   1/1     1            1           60d
~~~

That's it! I hope this clears how you can work with client certificates in OpenShift 4.