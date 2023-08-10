---
title:  "Integrating cert-manager with CFSSL Multirootca"
author: "Mario"
tags: [ "pki", "private ca", "TLS", "cfssl", "multirootca", "cert-manager" ]
url: "/integrating-cert-manager-with-cfssl-multirootca"
draft: false
date: 2023-08-10
#lastmod: 2023-08-10
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Integrating cert-manager with CFSSL Multirootca

In a [previous post](https://linuxera.org/pki-with-cfssl/) we saw how we could run our own PKI using the [CFSSL](https://github.com/cloudflare/cfssl) tooling. This post assumes you have read the previous one.

The starting point is an empty Kubernetes cluster, we want to deploy cert-manager on it and on top of that we want to get it configured to issue certificates with our own PKI infrastructure running Multirootca.

I’ll be using a Kubernetes v1.27 (latest at the time of this writing). The tool used to create the cluster is [kcli](https://kcli.readthedocs.io/) and the command used was:

~~~sh
kcli create kube generic -P ctlplanes=1 -P workers=1 -P ctlplane_memory=4096 -P numcpus=8 -P worker_memory=8192 -P image=fedora37 -P sdn=calico -P version=1.27 -P domain=linuxera.org cert-manager-cluster
~~~

## Introduction to cert-manager

I recommend reading the [official introduction](https://cert-manager.io/docs/) from the `cert-manager` project page.

We are going to configure `cert-manager` to talk to our `multirootca` server in order to request certificates. The external provider we will be using is the one created by Wikimedia [here](https://gerrit.wikimedia.org/r/plugins/gitiles/operations/software/cfssl-issuer/).

You can read more about external providers [here](https://cert-manager.io/docs/configuration/external/).

## Deploying cert-manager

We will be using `helm` to deploy `cert-manager`, the official steps are documented [here](https://cert-manager.io/docs/installation/helm/).

1. Add the helm repository:

    ~~~sh
    helm repo add jetstack https://charts.jetstack.io
    helm repo update jetstack
    ~~~

2. Deploy `cert-manager`:

    ~~~sh
    helm install \
      cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --create-namespace \
      --version v1.12.3 \
      --set installCRDs=true
    ~~~

3. If the deployment went well, we should have `cert-manager` running in our cluster:

    ~~~sh
    kubectl -n cert-manager get pods
    ~~~

    ~~~console
    NAME                                       READY   STATUS    RESTARTS   AGE
    cert-manager-875c7579b-qx62m               1/1     Running   0          2m16s
    cert-manager-cainjector-7bb6786867-q4t4x   1/1     Running   0          2m16s
    cert-manager-webhook-89dc55877-flj7g       1/1     Running   0          2m16s
    ~~~

## Integrating cert-manager with multirootca

As we mentioned earlier, we will be using the [Wikimedia CFSSL issuer external provider](https://gerrit.wikimedia.org/r/plugins/gitiles/operations/software/cfssl-issuer/).

Let's start by deploying the external provider into our cluster.

1. Add the helm repository:

    ~~~sh
    helm repo add wikimedia-charts https://helm-charts.wikimedia.org/stable
    helm repo update wikimedia-charts
    ~~~

2. Deploy the required CRDs:

    ~~~sh
    helm install \
      cfssl-issuer-crds wikimedia-charts/cfssl-issuer-crds
    ~~~

3. Deploy the external provider controller:

    ~~~sh
    helm install \
      cfssl-issuer wikimedia-charts/cfssl-issuer \
      --namespace cert-manager
    ~~~

4. A new pod should be running in the `cert-manager` namespace:

    ~~~sh
    kubectl -n cert-manager get pods -l app.kubernetes.io/name=cfssl-issuer
    ~~~

    ~~~console
    NAME                            READY   STATUS    RESTARTS   AGE
    cfssl-issuer-64f564f78f-gq4n9   1/1     Running   0          29s
    ~~~

## Configuring the CFSSL `ClusterIssuer`

Now that we have the external provider running, we will go ahead and configure a `ClusterIssuer` that will be available to all namespaces to request certificates. You can read more on `Issuers` [here](https://cert-manager.io/docs/concepts/issuer/).

1. Create a secret with the `auth key` required by the `multirootca` to sign the certificate requests.

    ~~~sh
    kubectl -n cert-manager create secret generic \
      cfssl-linuxera-internal-ca-key --from-literal=key=b50ed348c4643d34706470f36a646fd4
    ~~~

2. Since the certificate that exposes our multirootca server has been signed with an unknown CA to our Kubernetes cluster, request by cert-manager will fail due to untrusted CA. We have two options to get this sorted out: First option would be adding our intermediate ca to the Kubernetes cluster trusted CA bundle, the second option (and the one I'll be using) is mounting the intermediate CA inside the external provider controller pod.

    {{<tip>}}
The `intermediate-ca.pem` is a file that contains the CA certificate for our intermediate CA that will sign the certificates.
    {{</tip>}}

    1. Create a ConfigMap with the internal CA certificate.

        ~~~sh
        kubectl -n cert-manager create configmap \
          internal-ca-chain --from-file=ca-bundle.crt=/path/to/intermediate-ca.pem
        ~~~

    2. Patch the `cfssl-issuer` deployment to mount this ConfigMap

        ~~~sh
        kubectl -n cert-manager patch deployment \
          cfssl-issuer -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"cfssl-issuer"}],"containers":[{"name":"cfssl-issuer","volumeMounts":[{"mountPath":"/etc/pki/tls/certs/","name":"internal-ca-chain"}]}],"volumes":[{"configMap":{"name":"internal-ca-chain"},"name":"internal-ca-chain"}]}}}}'
        ~~~

3. Next, create the issuer. Here we define how to reach the `multirootca` server.

    ~~~sh
    cat <<EOF | kubectl apply -f -
    apiVersion: cfssl-issuer.wikimedia.org/v1alpha1
    kind: ClusterIssuer
    metadata:
      name: cfssl-internal-linuxera-ca
    spec:
      authSecretName: "cfssl-linuxera-internal-ca-key"
      bundle: false
      label: "linuxeraintermediate"
      profile: "host"
      url: "https://multirootca-server.linuxera.org:8000"
    EOF
    ~~~

4. We can check the Issuer status:

    ~~~sh
    kubectl get clusterissuer.cfssl-issuer cfssl-internal-linuxera-ca -o yaml
    ~~~

    ~~~console
    status:
      conditions:
      - lastTransitionTime: "2023-08-10T13:39:31Z"
        message: Success
        reason: cfssl-issuer.IssuerController.Reconcile
        status: "True"
        type: Ready
    ~~~

## Requesting Certificates via `cert-manager`

At this point the `ClusterIssuer` is ready and we can request certificates to be signed by our `multirootca` instance from Kubernetes via cert-manager. Let's see how.

1. Create a `Certificate` request.

    ~~~sh
    cat <<EOF | kubectl -n default apply -f -
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: test-host-cert-linuxera
    spec:
      secretName: test-host-cert-linuxera
      duration: 2160h # 90d
      renewBefore: 360h # 15d
      subject:
        organizations:
          - Linuxera Internal
      commonName: testhost-certmanager.linuxera.org
      isCA: false
      privateKey:
        algorithm: RSA
        encoding: PKCS1
        size: 2048
      usages:
        - server auth
      dnsNames:
        - testhost-certmanager.linuxera.org
      uris:
        - spiffe://cluster.local/ns/default/sa/default
      ipAddresses:
        - 192.168.122.50
      issuerRef:
        name: cfssl-internal-linuxera-ca
        kind: ClusterIssuer
        group: cfssl-issuer.wikimedia.org
    EOF
    ~~~

2. We will get our cert issued and stored in a secret.

    ~~~sh
    kubectl -n default get secret test-host-cert-linuxera -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -subject -issuer -startdate -enddate
    ~~~

    ~~~console
    subject=O = Linuxera Internal, CN = testhost-certmanager.linuxera.org
    issuer=C = ES, ST = Valencia, L = Valencia, O = Linuxera Internal, OU = Linuxera Internal Intermediate CA, CN = Linuxera Intermediate CA
    notBefore=Aug 10 13:41:00 2023 GMT
    notAfter=Aug  9 13:41:00 2024 GMT
    ~~~

## Useful Resources

- [https://cert-manager.io/docs/configuration/](https://cert-manager.io/docs/configuration/)
- [https://cert-manager.io/docs/usage/certificate/](https://cert-manager.io/docs/usage/certificate/)
- [https://gerrit.wikimedia.org/r/plugins/gitiles/operations/software/cfssl-issuer/](https://gerrit.wikimedia.org/r/plugins/gitiles/operations/software/cfssl-issuer/)