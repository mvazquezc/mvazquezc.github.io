---
title:  "Capabilities and Seccomp Profiles on Kubernetes"
author: "Mario"
tags: [ "okd", "origin", "containers", "kubernetes", "capabilities", "securitycontext", "seccomp", "syscalls" ]
url: "/capabilities-seccomp-kubernetes/"
draft: false
date: 2021-04-01
#lastmod: 2021-04-01
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Capabilities and Seccomp Profiles on Kubernetes

In a [previous post](https://linuxera.org/container-security-capabilities-seccomp/) we talked about **Linux Capabilities** and **Secure Compute Profiles**, in this post we are going to see how we can leverage them on [Kubernetes](https://kubernetes.io/).

We will need a Kubernetes cluster, I'm going to use [kcli](https://github.com/karmab/kcli) in order to get one. Below command will deploy a Kubernetes cluster on VMs:

> **NOTE**: You can create a [parameters file](https://kcli.readthedocs.io/en/latest/index.html#create-a-parameters-yml) with the cluster configuration as well.

~~~sh
# Create a Kubernetes 1.20 cluster with 1 master and 1 worker using calico as SDN, nginx as ingress controller, metallb for loadbalancer services and CRI-O as container runtime
kcli create kube generic -P masters=1 -P workers=1  -P master_memory=4096 -P numcpus=2 -P worker_memory=4096 -P sdn=calico -P version=1.20 -P ingress=true -P ingress_method=nginx -P metallb=true -P engine=crio -P domain=linuxera.org caps-cluster
~~~

After a few moments we will get the `kubeconfig` for accessing our cluster:

~~~
Kubernetes cluster caps-cluster deployed!!!
INFO export KUBECONFIG=$HOME/.kcli/clusters/caps-cluster/auth/kubeconfig
INFO export PATH=$PWD:$PATH
~~~

We can start using it right away:

~~~sh
export KUBECONFIG=$HOME/.kcli/clusters/caps-cluster/auth/kubeconfig
kubectl get nodes
~~~

~~~
NAME                                 STATUS   ROLES                  AGE     VERSION
caps-cluster-master-0.linuxera.org   Ready    control-plane,master   8m19s   v1.20.5
caps-cluster-worker-0.linuxera.org   Ready    worker                 3m33s   v1.20.5
~~~

## Capabilities on Kubernetes

Capabilities on Kubernetes are configured for pods or containers via the [SecurityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/).

In the next scenarios we are going to see how we can configure different capabilities for our containers and how they behave depending on the user running our container.

We will be using a demo application that listens on a given port, by default the application image uses a non-root user. In a [previous post](https://linuxera.org/container-security-capabilities-seccomp/) we mentioned how capabilities behave differently depending on the user that runs the process, we will see how that affects when running on containers.

### Container Runtime Default Capabilities

As previously mentioned, container runtimes come with a set of enabled capabilities that will be assigned to every container if not otherwise specified. We're using CRI-O in our Kubernetes cluster and we can find the default capabilities in the CRI-O configuration file at `/etc/crio/crio.conf` present in the nodes:

~~~
default_capabilities = [
	"CHOWN",
	"DAC_OVERRIDE",
	"FSETID",
	"FOWNER",
	"SETGID",
	"SETUID",
	"SETPCAP",
	"NET_BIND_SERVICE",
	"KILL",
]
~~~

The capabilities in the list above will be the ones added to containers by default.

**Pod running with root UID**

1. Create a namespace:

    ~~~sh
    NAMESPACE=test-capabilities
    kubectl create ns ${NAMESPACE}
    ~~~
2. Create a pod running our test application with UID 0:

    ~~~sh
    cat <<EOF | kubectl -n ${NAMESPACE} create -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: reversewords-app-captest-root
    spec:
      containers:
      - image: quay.io/mavazque/reversewords:ubi8
        name: reversewords
        securityContext:
          runAsUser: 0
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~
3. Review the capability sets for the application process:

    ~~~sh
    kubectl -n ${NAMESPACE} exec -ti reversewords-app-captest-root -- grep Cap /proc/1/status
    ~~~

    ~~~
    CapInh:	00000000000005fb
    CapPrm:	00000000000005fb
    CapEff:	00000000000005fb
    CapBnd:	00000000000005fb
    CapAmb:	0000000000000000
    ~~~
4. If we decode the `effective` set this is what we get:

    ~~~sh
    capsh --decode=00000000000005fb
    ~~~

    > **NOTE**: You can see how the pod got assigned the runtime's default caps.

    ~~~
    0x00000000000005fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service
    ~~~

**Pod running with non-root UID**

1. Create a pod running our test application with a `non-root` UID:

    ~~~sh
    NAMESPACE=test-capabilities
    cat <<EOF | kubectl -n ${NAMESPACE} create -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: reversewords-app-captest-nonroot
    spec:
      containers:
      - image: quay.io/mavazque/reversewords:ubi8
        name: reversewords
        securityContext:
          runAsUser: 1024
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~
2. Review the capability sets for the application process:

    ~~~sh
    kubectl -n ${NAMESPACE} exec -ti reversewords-app-captest-nonroot -- grep Cap /proc/1/status
    ~~~

    ~~~
    CapInh:	00000000000005fb
    CapPrm:	0000000000000000
    CapEff:	0000000000000000
    CapBnd:	00000000000005fb
    CapAmb:	0000000000000000
    ~~~

You can see how the `effective` and `permitted` sets were cleared. We explained that behaviour in our [previous post](https://linuxera.org/container-security-capabilities-seccomp/). That happens because we're doing `execve` to an unprivileged process so those capability sets get cleared.

This has some consequences when running our workloads on Kubernetes, outside Kubernetes we could use `Ambient` capabilities, but at the time of this writing, Ambient capabilities [are not supported on Kubernetes](https://github.com/kubernetes/kubernetes/issues/56374). This means that we can only use file capabilities or capability aware programs in order to get capabilities on programs running as nonroot on Kubernetes.

### Configuring capabilities for our workloads

At this point we know what are the differences with regards to capabilities when running our workloads with a `root` or a `nonroot` UID. In the next scenarios we are going to see how we can configure our workloads so they only get the required capabilities they need in order to run.

**Workload running with root UID**

1. Create a deployment for our workload:

    > **NOTE**:  We are dropping all of the runtime's default capabilities, on top of that we add the `NET_BIND_SERVICE` capability and request the app to run with root UID. In the environment variables we configure our app to listen on port 80.

    ~~~sh
    NAMESPACE=test-capabilities
    cat <<EOF | kubectl -n ${NAMESPACE} create -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        app: reversewords-app-rootuid
      name: reversewords-app-rootuid
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: reversewords-app-rootuid
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: reversewords-app-rootuid
        spec:
          containers:
          - image: quay.io/mavazque/reversewords:ubi8
            name: reversewords
            resources: {}
            env:
            - name: APP_PORT
              value: "80"
            securityContext:
              runAsUser: 0
              capabilities:
                drop:
                - CHOWN
                - DAC_OVERRIDE
                - FSETID
                - FOWNER
                - SETGID
                - SETUID
                - SETPCAP
                - KILL
                add:
                - NET_BIND_SERVICE
    status: {}
    EOF
    ~~~
2. We can check the logs for our application and see that it's working fine:

    ~~~sh
    kubectl -n ${NAMESPACE} logs deployment/reversewords-app-rootuid
    ~~~

    ~~~
    2021/04/01 09:59:39 Starting Reverse Api v0.0.18 Release: NotSet
    2021/04/01 09:59:39 Listening on port 80
    ~~~
3. If we look at the capability sets this is what we get:

    ~~~sh
    kubectl -n ${NAMESPACE} exec -ti deployment/reversewords-app-rootuid -- grep Cap /proc/1/status
    ~~~

    ~~~
    CapInh:	0000000000000400
    CapPrm:	0000000000000400
    CapEff:	0000000000000400
    CapBnd:	0000000000000400
    CapAmb:	0000000000000000
    ~~~
4. As expected, only `NET_BIND_SERVICE` capability is available:

    ~~~sh
    capsh --decode=0000000000000400
    ~~~

    ~~~
    0x0000000000000400=cap_net_bind_service
    ~~~

The workload worked as expected when running with `root` UID, in the next scenario we will try the same app but this time running with a `non-root` UID.

**Workload running with non-root UID**

1. Create a deployment for our workload:

    > **NOTE**:  We are dropping all of the runtime's default capabilities, on top of that we add the `NET_BIND_SERVICE` capability and request the app to run with non-root UID. In the environment variables we configure our app to listen on port 80.

    ~~~sh
    NAMESPACE=test-capabilities
    cat <<EOF | kubectl -n ${NAMESPACE} create -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        app: reversewords-app-nonrootuid
      name: reversewords-app-nonrootuid
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: reversewords-app-nonrootuid
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: reversewords-app-nonrootuid
        spec:
          containers:
          - image: quay.io/mavazque/reversewords:ubi8
            name: reversewords
            resources: {}
            env:
            - name: APP_PORT
              value: "80"
            securityContext:
              runAsUser: 1024
              capabilities:
                drop:
                - CHOWN
                - DAC_OVERRIDE
                - FSETID
                - FOWNER
                - SETGID
                - SETUID
                - SETPCAP
                - KILL
                add:
                - NET_BIND_SERVICE
    status: {}
    EOF
    ~~~
2. We can check the logs for our application and see if it's working:

    ~~~sh
    kubectl -n ${NAMESPACE} logs deployment/reversewords-app-nonrootuid
    ~~~

    ~~~
    2021/04/01 10:09:10 Starting Reverse Api v0.0.18 Release: NotSet
    2021/04/01 10:09:10 Listening on port 80
    2021/04/01 10:09:10 listen tcp :80: bind: permission denied
    ~~~
3. This time the application didn't bind to port 80, let's update the app configuration so it binds to port 8080 and then we will review the capability sets:

    ~~~sh
    # Patch the app so it binds to port 8080
    kubectl -n ${NAMESPACE} patch deployment reversewords-app-nonrootuid -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"reversewords"}],"containers":[{"$setElementOrder/env":[{"name":"APP_PORT"}],"env":[{"name":"APP_PORT","value":"8080"}],"name":"reversewords"}]}}}}'
    # Get capability sets
    kubectl -n ${NAMESPACE} exec -ti deployment/reversewords-app-nonrootuid -- grep Cap /proc/1/status
    ~~~

    ~~~
    CapInh:	0000000000000400
    CapPrm:	0000000000000000
    CapEff:	0000000000000000
    CapBnd:	0000000000000400
    CapAmb:	0000000000000000
    ~~~
4. We don't have the `NET_BIND_SERVICE` in the `effective` set, if you remember from our [previous post](https://linuxera.org/container-security-capabilities-seccomp/) we would need the capability in the `ambient` set in order for our application to work, but as we said Kubernetes still doesn't support ambient capabilities so our only option is make use of file capabilities.
5. We have created a new image for our application and our application binary now has the `NET_BIND_SERVICE` capability in the `effective` and `permitted` file capability sets. Let's update the deployment configuration.

    > **NOTE**: We configured the app to bind to port 80 and changed the container image with the one that has the required changes.

    ~~~sh
    kubectl -n ${NAMESPACE} patch deployment reversewords-app-nonrootuid -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"reversewords"}],"containers":[{"$setElementOrder/env":[{"name":"APP_PORT"}],"env":[{"name":"APP_PORT","value":"80"}],"image":"quay.io/mavazque/reversewords-captest:latest","name":"reversewords"}]}}}}'
    ~~~
6. We can check the logs for our application and see if it's working:

    ~~~sh
    kubectl -n ${NAMESPACE} logs deployment/reversewords-app-nonrootuid
    ~~~

    ~~~
    2021/04/01 10:18:42 Starting Reverse Api v0.0.21 Release: NotSet
    2021/04/01 10:18:42 Listening on port 80
    ~~~
7. This time the application was able to bind to port 80, let's review the capability sets:

    ~~~sh
    kubectl -n ${NAMESPACE} exec -ti deployment/reversewords-app-nonrootuid -- grep Cap /proc/1/status
    ~~~

    > **NOTE**: Since our application binary has the required capability in its file capability sets the process thread was able to gain that capability:
    ~~~
    CapInh:	0000000000000400
    CapPrm:	0000000000000400
    CapEff:	0000000000000400
    CapBnd:	0000000000000400
    CapAmb:	0000000000000000
    ~~~
8. We can check the file capability configured in our application binary:

    ~~~sh
    kubectl -n ${NAMESPACE} exec -ti deployment/reversewords-app-nonrootuid -- getcap /usr/bin/reverse-words
    ~~~

    ~~~
    /usr/bin/reverse-words = cap_net_bind_service+eip
    ~~~

## Seccomp Profiles on Kubernetes

In this scenario we're going to reuse the Secure Compute profile we created in the [previous post](https://linuxera.org/container-security-capabilities-seccomp#secure-compute-profiles-in-action).

### Configuring Seccomp Profiles on the cluster nodes

By default `Kubelet` will try to find the `seccomp` profiles in the `/var/lib/kubelet/seccomp/` path. This path can be configured in the kubelet config.

We are going to create the two seccomp profiles that we will be using in the nodes.

Create below file on every node that can run workloads as `/var/lib/kubelet/seccomp/centos8-ls.json`:

> **NOTE**: This is the seccomp profile that allows us to run a `centos8` image that runs `ls /` as we saw in the previous post.

~~~json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64"
  ],
  "syscalls": [
    {
      "names": [
        "access",
        "arch_prctl",
        "brk",
        "capget",
        "capset",
        "chdir",
        "close",
        "epoll_ctl",
        "epoll_pwait",
        "execve",
        "exit_group",
        "fchown",
        "fcntl",
        "fstat",
        "fstatfs",
        "futex",
        "getdents64",
        "getpid",
        "getppid",
        "ioctl",
        "mmap",
        "mprotect",
        "munmap",
        "nanosleep",
        "newfstatat",
        "openat",
        "prctl",
        "pread64",
        "prlimit64",
        "read",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "sched_yield",
        "seccomp",
        "set_robust_list",
        "set_tid_address",
        "setgid",
        "setgroups",
        "setuid",
        "stat",
        "statfs",
        "tgkill",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW",
      "args": [],
      "comment": "",
      "includes": {},
      "excludes": {}
    }
  ]
}
~~~

### Configuring seccomp profiles for our workloads

1. Create a namespace:

    ~~~sh
    NAMESPACE=test-seccomp
    kubectl create ns ${NAMESPACE}
    ~~~
2. Seccomp profiles can be configured at pod or container level, this time we're going to configure it at pod level:

    > **NOTE**: We configured the seccompProfile `centos8-ls.json`.

    ~~~sh
    cat <<EOF | kubectl -n ${NAMESPACE} create -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: seccomp-ls-test
    spec:
      securityContext:
        seccompProfile:
          type: Localhost
          localhostProfile: centos8-ls.json
      containers:
      - image: registry.centos.org/centos:8
        name: seccomp-ls-test
        command: ["ls", "/"]
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~
3. The pod was executed with no issues:

    ~~~sh
    kubectl -n ${NAMESPACE} logs seccomp-ls-test
    ~~~

    ~~~
    bin
    dev
    ...
    ~~~
4. Let's try to create a new pod that runs `ls -l` instead. On top of that we will configure the seccomp profile at the container level.

    ~~~sh
    cat <<EOF | kubectl -n ${NAMESPACE} create -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: seccomp-lsl-test
    spec:
      containers:
      - image: registry.centos.org/centos:8
        name: seccomp-lsl-test
        command: ["ls", "-l", "/"]
        securityContext:
          seccompProfile:
            type: Localhost
            localhostProfile: centos8-ls.json
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~
5. As expected, the pod failed since the seccomp profile doesn't have all the required syscalls required for the command to run permitted:

    ~~~sh
    kubectl -n ${NAMESPACE} logs seccomp-lsl-test
    ~~~

    ~~~
    ls: cannot access '/': Operation not permitted
    ~~~

# Closing Thoughts

At this point you should've a clear understanding of when your workloads will benefit from using capabilities or seccomp profiles.

We've not been through how we can control which capabilities / seccomp a specific user can use, `PodSecurityPolicies` can be used to control such things on Kubernetes. In [OpenShift](openshift.com) you can use `SecurityContextConstraints`.

If you want to learn more around these topics feel free to take a look at the following SCCs lab: https://github.com/mvazquezc/scc-fun/blob/main/README.md
