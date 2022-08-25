---
title:  "Working with Pod Security Standards"
author: "Mario"
tags: [ "kubernetes", "k8s", "security", "capabilities", "securitycontext", "admission" ]
url: "/working-with-pod-security-standards/"
draft: false
date: 2022-08-24
#lastmod: 2022-08-24
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Working with Pod Security Standards

In Kubernetes v1.25 Pod Security admission has moved to stable, replacing Pod Security Policy admission. This feature has been in beta and enabled by default since Kubernetes v1.23 in this post we are going to cover what's new with Pod Security Admission (PSA) and how it affects the workloads being deployed in our clusters.

{{<tip>}}
For this post I'll be running a Kubernetes v1.25 cluster. If you want to try this in your own environment you can use your favorite tool to get a K8s cluster up and running, I'll be using [kcli](https://github.com/karmab/kcli).
{{</tip>}}

~~~sh
# Create a Kubernetes 1.25 cluster with 1 master and 1 worker using calico as SDN, nginx as ingress controller, metallb for loadbalancer services and CRI-O as container runtime
kcli create kube generic -P masters=1 -P workers=1  -P master_memory=4096 -P numcpus=2 -P worker_memory=4096 -P sdn=calico -P version=1.25 -P ingress=true -P ingress_method=nginx -P metallb=true -P engine=crio -P domain=linuxera.org psa-cluster
~~~

This is how our cluster looks like:

~~~sh
kubectl get nodes

NAME                                STATUS   ROLES                  AGE     VERSION
psa-cluster-master-0.linuxera.org   Ready    control-plane,master   4m19s   v1.24.4
psa-cluster-worker-0.linuxera.org   Ready    worker                 1m20s   v1.24.4
~~~

## Pod Security Admission

The Pod Security Admission relies on both Pod Security Standards which define the different security policies that need to be checked for workloads and Pod Admission Modes that define how the standards are applied for a given namespace.

### Pod Security Standards

This new admission plugin relies on pre-backed [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards). These standards will evolve every Kubernetes release to include / adapt new security rules.

As of Kubernetes v1.25 there are three Pod Security Standards defined:

{{<tip>}}
You can read each standard requirements on [this link](https://kubernetes.io/docs/concepts/security/pod-security-standards).
{{</tip>}}

* `privileged`
* `baseline`
* `restricted`

### Pod Admission Modes

The cluster admin/namespace admin can configure an admission mode that will be used to do admission validations against workloads being deployed in the namespace. There are three admission modes that can be configured on a namespace:

* `enforce`: Policy violations will cause the pod to be rejected.
* `audit`: Policy violations will be logged in the audit log, pod will be allowed.
* `warn`: Policy violations will case a user-facing warning, pod will be allowed.

Each mode can be configured with a different Pod Security Standard. For example, a namespace could enforce using the `privileged` standard and audit/warn via the`restricted` standard.

The admission modes and the standards to be used are configured at the namespace level via the use of the `pod-security.kubernetes.io/<MODE>: <LEVEL>` label.

As earlier mentioned, these Pod Security Standards will evolve over time, and since these are versioned we can specify which version of a specific mode we want to enforce via the use of the `pod-security.kubernetes.io/<MODE>-version: <VERSION>` label, where \<VERSION\> refers to a Kubernetes minor version like `v1.25`.

If we put all this information together, we can get to a namespace definition like the one below:

{{<tip>}}
In the example below we use the version `v1.25`, a namespace could also point to the latest available by using `latest` instead.
{{</tip>}}

~~~yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test-namespace
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: v1.25
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.25
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.25
~~~

It's important to mention that audit and warning modes are applied to workload resources (resources that have a pod template definition) like Deployments, Jobs, etc. to help catch violations early. On the other hand, enforce mode is applied to the resulting pod object.

### Pod Security Admission Configuration

Pod Security Admission comes pre-configured in Kubernetes v1.25 with the least restrictive policy, it's possible to modify the default configuration by modifying the admission configuration for this plugin, you can read [here](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/#configure-the-admission-controller) how to do it.

If you checked the link above, you have seen that exemptions can be configured for the admission, this will allow the cluster admin to configure users, runtime classes or namespaces that won't be evaluated by PSA. From this three exemptions, the runtime class could be helpful if you want to keep a namespace as restrictive as possible by default, but then have some workload that is not evaluated against a PSA.

## Pod Security Standards in Action

Now that we know the basics around PSA, we can go ahead and run some tests to understand how it works. We will be using a [simple go app](https://github.com/mvazquezc/reverse-words) that exposes a service on a port of our choice.

### Non-restrictive namespace

In this first example we're going to deploy our workload in a namespace that enforces the `privileged` standard and audits/warns the `restricted` standard.

1. Create the namespace for our workload with the appropriated PSA settings:

    ~~~sh
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: Namespace
    metadata:
      name: non-restrictive-namespace
      labels:
        pod-security.kubernetes.io/enforce: privileged
        pod-security.kubernetes.io/enforce-version: v1.25
        pod-security.kubernetes.io/audit: restricted
        pod-security.kubernetes.io/audit-version: v1.25
        pod-security.kubernetes.io/warn: restricted
        pod-security.kubernetes.io/warn-version: v1.25
    EOF
    ~~~

2. Create the workload:

    ~~~sh
    cat <<EOF | kubectl -n non-restrictive-namespace apply -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: go-app
      name: go-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: go-app
      strategy: {}
      template:
        metadata:
          labels:
            app: go-app
        spec:
          containers:
          - image: quay.io/mavazque/reversewords:latest
            name: reversewords
            resources: {}
    EOF
    ~~~

We got some client warnings (caused by the warn mode) saying the violations of our workload when checked against the `restricted` standard:

~~~sh
Warning: would violate PodSecurity "restricted:v1.25": allowPrivilegeEscalation != false (container "reversewords" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "reversewords" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "reversewords" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "reversewords" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
~~~

But the workload is running:

~~~sh
kubectl -n non-restrictive-namespace get pod

NAME                      READY   STATUS    RESTARTS   AGE
go-app-5b954b7b74-kwkwn   1/1     Running   0          1m30s
~~~

In the next scenario we will configure the enforce mode to the restricted standard.

### Restrictive namespace

1. Create the namespace for our workload with the appropriated PSA settings:

    ~~~sh
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: Namespace
    metadata:
      name: restrictive-namespace
      labels:
        pod-security.kubernetes.io/enforce: restricted
        pod-security.kubernetes.io/enforce-version: v1.25
        pod-security.kubernetes.io/audit: restricted
        pod-security.kubernetes.io/audit-version: v1.25
        pod-security.kubernetes.io/warn: restricted
        pod-security.kubernetes.io/warn-version: v1.25
    EOF
    ~~~

2. Create the workload:

    ~~~sh
    cat <<EOF | kubectl -n restrictive-namespace apply -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: go-app
      name: go-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: go-app
      strategy: {}
      template:
        metadata:
          labels:
            app: go-app
        spec:
          containers:
          - image: quay.io/mavazque/reversewords:latest
            name: reversewords
            resources: {}
    EOF
    ~~~

Again, we got some client warnings (caused by the warn mode) saying the violations of our workload when checked against the `restricted` standard:

~~~sh
Warning: would violate PodSecurity "restricted:v1.25": allowPrivilegeEscalation != false (container "reversewords" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "reversewords" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "reversewords" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "reversewords" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
~~~

And this time, the workload is NOT running:

~~~sh
kubectl -n restrictive-namespace get pod

No resources found in restrictive-namespace namespace.
~~~

If you remember, the enforce mode is applied against the pod object and not against the workload objects (like Deployment in this case). That's why the deployment was admitted but the pod it's not.

We can see in the namespace events / replicaset status why the pod is not running:

~~~sh
kubectl -n restrictive-namespace get events

LAST SEEN   TYPE      REASON              OBJECT                         MESSAGE
3m44s       Warning   FailedCreate        replicaset/go-app-5b954b7b74   Error creating: pods "go-app-5b954b7b74-dfq9g" is forbidden: violates PodSecurity "restricted:v1.25": allowPrivilegeEscalation != false (container "reversewords" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "reversewords" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "reversewords" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "reversewords" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
~~~

If we want this workload to be admitted in the cluster we need to fine tune the pod's configuration, let's remove the deployment and get it created with a config allowed by the `restricted` standard.

1. Remove the deployment

    ~~~sh
    kubectl -n restrictive-namespace delete deployment go-app
    ~~~

2. Create the workload with the proper config:

    ~~~sh
    cat <<EOF | kubectl -n restrictive-namespace apply -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: go-app
      name: go-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: go-app
      strategy: {}
      template:
        metadata:
          labels:
            app: go-app
        spec:
          containers:
          - image: quay.io/mavazque/reversewords:latest
            name: reversewords
            resources: {}
            securityContext:
              allowPrivilegeEscalation: false
              capabilities:
                drop:
                  - ALL
              runAsNonRoot: true
              runAsUser: 1024
              seccompProfile:
                type: RuntimeDefault
    EOF
    ~~~

This time we didn't get any warnings and if we check for pods in the namespace we will see our workload is running:

~~~sh
kubectl -n restrictive-namespace get pod

NAME                      READY   STATUS    RESTARTS   AGE
go-app-5f45c655b6-z26kv   1/1     Running   0          25s
~~~

### Tip 1 - Check if a given workload would be rejected in a given namespace

You can try to create a workload against a given namespace in dry-run mode and get client warnings, example:

~~~sh
cat <<EOF | kubectl -n restrictive-namespace apply --dry-run=server -f - 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: go-app
  name: go-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-app
  strategy: {}
  template:
    metadata:
      labels:
        app: go-app
    spec:
      containers:
      - image: quay.io/mavazque/reversewords:latest
        name: reversewords
        resources: {}
EOF
~~~

You will get a warning like this:

~~~sh
Warning: would violate PodSecurity "restricted:v1.25": allowPrivilegeEscalation != false (container "reversewords" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "reversewords" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "reversewords" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "reversewords" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/go-app created (server dry run)
~~~

### Tip 2 - Check if workloads on a given namespace would violate a given policy

You can try to label a namespace in dry-run mode and get client warnings, as an example let's see what would happen if we moved the namespace from the first scenario from the `privileged` standard to the `restricted` one:

~~~sh
kubectl label --dry-run=server --overwrite ns non-restrictive-namespace pod-security.kubernetes.io/enforce=restricted
~~~

You will get a warning like this:

~~~sh
Warning: existing pods in namespace "non-restrictive-namespace" violate the new PodSecurity enforce level "restricted:v1.25"
Warning: go-app-5b954b7b74-kwkwn: allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true, seccompProfile
namespace/non-restrictive-namespace labeled
~~~

## Closing Thoughts

Pod Security Admission is a great addition to the Kubernetes security, I hope this time its adoption increases compared to PSPs. In the next post we will talk about the new changes around Seccomp that were introduced in Kubernetes.