---
title:  "CPU and Memory Management on Kubernetes with Cgroupsv2"
author: "Mario"
tags: [ "kubernetes", "openshift", "cgroups", "cgroupsv2" ]
url: "/cpu-memory-management-kubernetes-cgroupsv2"
draft: false
date: 2023-04-06
#lastmod: 2023-04-19
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# CPU and Memory Management on Kubernetes with Cgroupsv2

In this post I'll try to explain how CPU and Memory management works under the hood on Kubernetes. If you ever wondered what happens when you set `requests` and `limits` for your pods, keep reading!

{{<attention>}}
This is the result of my exploratory work around cgroupsv2 and their application to Kubernetes. Even though I tried really hard to make sure the information in this post is accurate, I'm far from being an expert on the topic and some information may not be 100% accurate. If you detect something that is missing / wrong, please comment the post and I'll fix it!
{{</attention>}}

I'll be using a Kubernetes v1.26 (latest at the time of this writing) with an operating system with support for cgroupsv2 like Fedora 37. The tool used to create the cluster is [kcli](https://kcli.readthedocs.io/) and the command used was:

~~~sh
kcli create kube generic -P ctlplanes=1 -P workers=1 -P ctlplane_memory=4096 -P numcpus=8 -P worker_memory=8192 -P image=fedora37 -P sdn=calico -P version=1.26 -P ingress=true -P ingress_method=nginx -P metallb=true -P domain=linuxera.org resource-mgmt-cluster
~~~

## Introduction to Cgroups

As we explained in a [previous post](https://linuxera.org/containers-under-the-hood/), Cgroups are used to limit what resources are available to containers on the system. In Kubernetes it's not different.

Cgroups version 2 introduces improvements and new features on top of Cgroups version 1, you can read more about what changed in [this link](https://man7.org/linux/man-pages/man7/cgroups.7.html#CGROUPS_VERSION_2).

In the next sections we will see how we can limit memory and cpu for processes.

### Limiting memory using Cgroupsv2

Limiting memory is pretty straightforward, we just set a memory.max and since the memory is a resource that cannot be compressed, once the process reaches the limit it will be killed.

We will be using this python script:

~~~sh
cat <<EOF > /opt/dumb.py
f = open("/dev/urandom", "r", encoding = "ISO-8859-1")
data = ""
i=0
while i < 20:
    data += f.read(10485760) # 10MiB
    i += 1
    print("Used %d MiB" % (i * 10))
EOF
~~~

1. Let's create a new cgroup under the system.slice:

    ~~~sh
    sudo mkdir -p /sys/fs/cgroup/system.slice/memorytest
    ~~~

2. Set a limit of 200MiB of RAM for this cgroup and disable swap:

    ~~~sh
    echo 200Mi > /sys/fs/cgroup/system.slice/memorytest/memory.max
    echo "0" > /sys/fs/cgroup/system.slice/memorytest/memory.swap.max
    ~~~

3. Add the current shell process to the cgroup:

    ~~~sh
    echo $$ > /sys/fs/cgroup/system.slice/memorytest/cgroup.procs
    ~~~

4. Run the python script:

    ~~~sh
    python3 /opt/dumb.py
    ~~~

    {{<attention>}}
Even if the script stopped at 80MB that's caused because the python interpreter + shared libraries consume also memory. We can check the current memory usage in the cgroup by using the `systemd-cgtop system.slice/memorytest` command or with something like this `MEMORY=$(cat /sys/fs/cgroup/system.slice/memorytest/memory.current);echo $(( $MEMORY / 1024 / 1024 ))MiB`
    {{</attention>}}

    ~~~console
    Used 10 MiB
    Used 20 MiB
    Used 30 MiB
    Used 40 MiB
    Used 50 MiB
    Used 60 MiB
    Used 70 MiB
    Used 80 MiB
    Killed
    ~~~

5. Remove the cgroup:

    {{<warning>}}
Make sure you closed the shell attached to the cgroup before running the command below, otherwise it will fail.
    {{</warning>}}

    ~~~sh
    sudo rmdir /sys/fs/cgroup/system.slice/memorytest/
    ~~~

Now that we have seen how to limit memory, let's see how to limit CPU.

### Limiting CPU using Cgroupsv2

Limiting CPU is not as straightforward as limiting memory, since CPU can be compressed we can make sure that a process doesn't use more CPU than allowed without having to kill it.

We need to configure the parent cgroup so it has the `cpu` and `cpuset` controllers enabled for its children groups. Below example configures the controllers for the `system.slice` cgroup which is the parent group we will be using. By default, only `memory` and `pids` controllers are enabled.

Enable `cpu` and `cpuset` controllers for the `/sys/fs/cgroup/` and `/sys/fs/cgroup/system.slice` children groups:

~~~sh
echo "+cpu" >> /sys/fs/cgroup/cgroup.subtree_control
echo "+cpuset" >> /sys/fs/cgroup/cgroup.subtree_control
echo "+cpu" >> /sys/fs/cgroup/system.slice/cgroup.subtree_control
echo "+cpuset" >> /sys/fs/cgroup/system.slice/cgroup.subtree_control
~~~

#### Limiting CPU — Pin process to CPU and limit CPU bandwidth

1. Let's create a new cgroup under the system.slice:

    ~~~sh
    sudo mkdir -p /sys/fs/cgroup/system.slice/cputest
    ~~~

2. Assign only 1 core to this cgroup

    {{<attention>}}
Below command assigns core 0 to our cgroup.
    {{</attention>}}

    ~~~sh
    echo "0" > /sys/fs/cgroup/system.slice/cputest/cpuset.cpus
    ~~~

3. Set a limit of half-cpu for this cgroup:

    {{<attention>}}
The value for cpu.max is in units of 1/1000ths of a CPU core, so 50000 represents 50% of a single core.
    {{</attention>}}

    ~~~sh
    echo 50000 > /sys/fs/cgroup/system.slice/cputest/cpu.max
    ~~~

4. Add the current shell process to the cgroup:

    ~~~sh
    echo $$ > /sys/fs/cgroup/system.slice/cputest/cgroup.procs
    ~~~

5. Download the cpuload utility:

    ~~~sh
    curl -L https://github.com/vikyd/go-cpu-load/releases/download/0.0.1/go-cpu-load-linux-amd64 -o /tmp/cpuload && chmod +x /tmp/cpuload
    ~~~

6. Run the cpu load:

    {{<attention>}}
We're requesting 1 core and 50% of the CPU, this should fit within the `cpu.max` setting.
    {{</attention>}}

    ~~~sh
    /tmp/cpuload -p 50 -c 1
    ~~~

7. If we check with `systemd-cgtop system.slice/cputest` the usage we will see something like this:

    ~~~console
    Control Group           Tasks   %CPU   Memory  Input/s Output/s
    system.slice/cputest        6   47.7   856.0K        -        -
    ~~~

8. Since we're within the budget, we shouldn't see any throttling happening:

    ~~~sh
    grep throttled /sys/fs/cgroup/system.slice/cputest/cpu.stat
    ~~~

    ~~~console
    nr_throttled 0
    throttled_usec 0
    ~~~

9. If we stop the cpuload command and request 100% of 1 core we will see throttling:

    ~~~sh
    /tmp/cpuload -p 100 -c 1
    ~~~

    ~~~console
    Control Group           Tasks   %CPU   Memory  Input/s Output/s
    system.slice/cputest        6   50.0   720.0K        -        -
    ~~~

    ~~~sh
    grep throttled /sys/fs/cgroup/system.slice/cputest/cpu.stat
    ~~~

    ~~~console
    nr_throttled 336
    throttled_usec 16640745
    ~~~

10. Remove the cgroup:

    {{<warning>}}
Make sure you closed the shell attached to the cgroup before running the command below, otherwise it will fail.
    {{</warning>}}

    ~~~sh
    sudo rmdir /sys/fs/cgroup/system.slice/cputest/
    ~~~

This use case is very simple, we pinned our process to 1 core and limited the CPU to half a core. Let's see what happens when multiple processes compete for the CPU.

#### Limiting CPU — Pin processes to CPU and limit CPU bandwidth

1. Let's create a new cgroup under the system.slice:

    ~~~sh
    sudo mkdir -p /sys/fs/cgroup/system.slice/compitingcputest
    ~~~

2. Assign only 1 core to this cgroup

    {{<attention>}}
Below command assigns core 0 to our cgroup.
    {{</attention>}}

    ~~~sh
    echo "0" > /sys/fs/cgroup/system.slice/compitingcputest/cpuset.cpus
    ~~~

3. Set a limit of one cpu for this cgroup:

    {{<attention>}}
The value for cpu.max is in units of 1/1000ths of a CPU core, so 100000 represents 100% of a single core.
    {{</attention>}}

    ~~~sh
    echo 100000 > /sys/fs/cgroup/system.slice/compitingcputest/cpu.max
    ~~~

4. Open two shells and attach their process to the cgroup:

    ~~~sh
    echo $$ > /sys/fs/cgroup/system.slice/compitingcputest/cgroup.procs
    ~~~

5. Run the cpu load in one of the shells:

    {{<attention>}}
We're requesting 1 core and 100% of the CPU, this should fit within the `cpu.max` setting.
    {{</attention>}}

    ~~~sh
    /tmp/cpuload -p 100 -c 1
    ~~~

6. If we check for throttling we will see that no throttling is happening.

    ~~~sh
    grep throttled /sys/fs/cgroup/system.slice/compitingcputest/cpu.stat
    ~~~

    ~~~console
    nr_throttled 0
    throttled_usec 0
    ~~~

7. Run another instance of cpuload on the other shell:

    ~~~sh
    /tmp/cpuload -p 100 -c 1
    ~~~

8. At this point, we shouldn't see throttling, but the CPU time would be shared by the two processes, in the `top` output below we can see that each process is consuming half cpu.

    ~~~console
    PID    USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                         
    822742 root      20   0    4104   2004   1680 S  49.8   0.0   0:24.30 cpuload                                         
    822717 root      20   0    4104   2008   1680 S  49.5   0.1   6:28.51 cpuload             
    ~~~

9. Close the shells and remove the cgroup:

    {{<warning>}}
Make sure you closed the shell attached to the cgroup before running the command below, otherwise it will fail.
    {{</warning>}}

    ~~~sh
    sudo rmdir /sys/fs/cgroup/system.slice/compitingcputest/
    ~~~

In this use case, we pinned our process to 1 core and limited the CPU to one core. On top of that, we spawned two processes that competed for CPU. Since CPU bandwidth distribution was not set, each process got half cpu. In the next section we will see how to distribute CPU across processes using weights.

#### Limiting CPU — Pin processes to CPU, limit and distribute CPU bandwidth

1. Let's create a new cgroup under the system.slice with two sub-groups (appA and appB):

    ~~~sh
    sudo mkdir -p /sys/fs/cgroup/system.slice/distributedbandwidthtest/{appA,appB}
    ~~~

2. Enable `cpu` and `cpuset` controllers for the `/sys/fs/cgroup/system.slice/distributedbandwidthtest` children groups:

    ~~~sh
    echo "+cpu" >> /sys/fs/cgroup/system.slice/distributedbandwidthtest/cgroup.subtree_control
    echo "+cpuset" >> /sys/fs/cgroup/system.slice/distributedbandwidthtest/cgroup.subtree_control
    ~~~

3. Assign only 1 core to the parent cgroup

    {{<attention>}}
Below command assigns core 0 to our cgroup.
    {{</attention>}}

    ~~~sh
    echo "0" > /sys/fs/cgroup/system.slice/distributedbandwidthtest/cpuset.cpus
    ~~~

4. Set a limit of one cpu for this cgroup:

    {{<attention>}}
The value for cpu.max is in units of 1/1000ths of a CPU core, so 100000 represents 100% of a single core.
    {{</attention>}}

    ~~~sh
    echo 100000 > /sys/fs/cgroup/system.slice/distributedbandwidthtest/cpu.max
    ~~~

5. Open two shells and attach their process to the different child cgroups, then run cpuload:

    1. Shell 1

        ~~~sh
        echo $$ > /sys/fs/cgroup/system.slice/distributedbandwidthtest/appA/cgroup.procs
        /tmp/cpuload -p 100 -c 1
        ~~~

    2. Shell 2

        ~~~sh
        echo $$ > /sys/fs/cgroup/system.slice/distributedbandwidthtest/appB/cgroup.procs
        /tmp/cpuload -p 100 -c 1
        ~~~

6. If you check the top output, you will see that CPU is evenly distributed across both processes, let's modify weights to give more CPU to appB cgroup.

7. In cgroupvs1 there was `cpu shares` concept, in cgroupsv2 this changed and now we use `cpu weights`. All weights are in the range [1, 10000] with the default at 100. This allows symmetric multiplicative biases in both directions at fine enough granularity while staying in the intuitive range. If we wanted to give `appA` a `30%` of the CPU and `appB` the other `70%`, providing that the parent cgroup CPU weight is set to 100 this is the configuration we will apply:

    ~~~sh
    cat /sys/fs/cgroup/system.slice/distributedbandwidthtest/cpu.weight
    ~~~

    ~~~console
    100
    ~~~

    1. Assign 30% of the cpu to appA

        ~~~sh
        echo 30 > /sys/fs/cgroup/system.slice/distributedbandwidthtest/appA/cpu.weight
        ~~~

    2. Assign 70% of the cpu to appB

        ~~~sh
        echo 70 > /sys/fs/cgroup/system.slice/distributedbandwidthtest/appB/cpu.weight
        ~~~

8. If we look at the top output we will see something like this:

    {{<attention>}}
You can see how one of the cpuload processes is getting 70% of the cpu while the other is getting the other 30%.
    {{</attention>}}

    ~~~console
    PID    USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                 
    1077   root      20   0    4104   2008   1680 S  70.0   0.1  12:41.27 cpuload                                                                                 
    1071   root      20   0    4104   2008   1680 S  30.0   0.1  12:24.14 cpuload 
    ~~~

9. Close the shells and remove the cgroups:

    {{<warning>}}
Make sure you closed the shell attached to the cgroup before running the command below, otherwise it will fail.
    {{</warning>}}

    ~~~sh
    sudo rmdir /sys/fs/cgroup/system.slice/distributedbandwidthtest/appA/
    sudo rmdir /sys/fs/cgroup/system.slice/distributedbandwidthtest/appB/
    sudo rmdir /sys/fs/cgroup/system.slice/distributedbandwidthtest/
    ~~~

At this point, we should have a clear understanding on how the basics work, next section will introduce these concepts applied to Kubernetes.

## Resource Management on Kubernetes

We won't be covering the basics, I recommend reading the [official docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/). We will be focusing on CPU/Memory requests and limits.

### Cgroupsv2 configuration for a Kubernetes node

Before describing the cgroupsv2 configuration, we need to understand how Kubelet configurations will impact cgroupsv2 configurations. In our test cluster, we have the following Kubelet settings in order to reserve resources for system daemons:

~~~yaml
systemReserved:
  cpu: 500m
  memory: 500Mi
~~~

If we describe the node this is what we will see:

~~~sh
oc describe node <compute-node>
~~~

{{<attention>}}
You can see how half cpu (500m) and 500Mi of memory have been subtracted from the allocatable capacity.
{{</attention>}}

~~~console
Capacity:
  cpu:                4
  <omitted>
  memory:             6069552Ki
Allocatable:
  cpu:                3500m
  <omitted>
  memory:             5455152Ki
~~~

Even if we remove resources from the allocatable capacity, depending on the QoS of our pods we would be able to over commit on resources, at that point, eviction may happen and cgroups will make sure that pods with more priority get the required resources they asked for.

#### Cgroupsv2 configuration on the node

In a regular Kubernetes node we will have at least three main parent cgroups:

* `kubepods.slice`: Parent cgroup used by Kubernetes to place pod processes. It has two child cgroups named after pod QoS inside: `kubepods-besteffort.slice` and `kubepods-burstable.slice`. Guaranteed pods get created inside this parent cgroup.
* `system.slice`: Parent cgroup used by the O.S to place system processes. Kubelet, sshd, etc. run here.
* `user.slice`: Parent cgroup used by the O.S to place user processes. When you run a regular command, it runs here.

{{<attention>}}
The output below omits certain directories. It's meant to help visualize the description above.
{{</attention>}}

~~~console
/sys/fs/cgroup/
├── kubepods.slice
│   ├── kubepods-besteffort.slice
│   └── kubepods-burstable.slice
├── system.slice
│   ├── kubelet.service
│   └── sshd.service
└── user.slice
    └── user-1000.slice
~~~

In the previous sections we have talked about how _`cpu.weight`_ works for distributing CPU bandwidth to processes. The parent cgroups in a Kubernetes node will be configured as follows:

* `system.slice`: A _cpu.weight_ of `100`.
* `user.slice`: A _cpu.weight_ of `100`.

In a Kubernetes node, we won't have much/any user processes running. So at the end, the two cgroups competing for resources will be `system.slice` and `kubepods.slice`. But wait, what _cpu.weight_ is configured for `kubepods.slice`?

When Kubelet starts it detects the number of CPUs available on the node, on top of that it reads the `systemReserved.cpu` configuration. That will give you a number of milicores available for Kubernetes to use on that node.

For example, if I have a 4 CPU node that's 4000 milicores, if I reserved 500m for the system resources (kubelet, sshd, etc.) that leaves Kubernetes with 3500 milicores that can be assigned to workloads.

Now, Kubelet knows that 3500 milicores is the amount of CPU that can be _assigned_ to workloads (and assigned means that is more or less assured in case workloads request it). The cgroups _cpu.weight_ needs to be configured so CPU get distributed accordingly, let's see how that's done:

1. In the past (cgroupsv1), CPU Shares were used and every CPU was represented by 1024 Shares. Now, we need to translate from shares to weight and the community has a formula for that (more info [here](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2254-cgroup-v2#phase-1-convert-from-cgroups-v1-settings-to-v2)).
2. In cgroupsv2 we still use Shares under the hood, but that's only because the formula created to not having to change the specification requires them. So we have a [constant](https://github.com/kubernetes/kubernetes/blob/release-1.27/pkg/kubelet/cm/helpers_linux.go#L45) that sets the Shares/CPU to 1024 and a function that [translates milicores to shares](https://github.com/kubernetes/kubernetes/blob/release-1.27/pkg/kubelet/cm/helpers_linux.go#L85).
3. Finally, there is a function that [translates CPU Shares to CPU Weight](https://github.com/kubernetes/kubernetes/blob/release-1.27/pkg/kubelet/cm/cgroup_manager_linux.go#L566) using the formula from 1.

After we know the weight that needs to be applied to the `kubepods.slice`, the relevant code that does that is [here](https://github.com/kubernetes/kubernetes/blob/release-1.27/pkg/kubelet/cm/node_container_manager_linux.go#L115) and [here](https://github.com/kubernetes/kubernetes/blob/release-1.27/pkg/kubelet/cm/cgroup_manager_linux.go#L435).

Continuing with the example, the _cpu.weight_ for our 4 CPU node with 500 milicores reserved for system resources would be:

Formula being used: (((cpuShares - 2) * 9999) / 262142) + 1

cpuShares = 3.5 Cores * 1024 = 3584

cpu.weight = (((3584 - 2) * 9999) / 262142) + 1 = 137,62

If we check our node:

~~~sh
cat /sys/fs/cgroup/kubepods.slice/cpu.weight
~~~

~~~console
137
~~~

At this point we know how the different cgroups get configured on the node, next let's see what happens when `kubepods.slice` and `system.slice` compete for cpu.

#### `kubepods.slice` and `system.slice` competing for CPU

In the previous section we have seen how the different cgroups get configured on our 4 CPU node, in this section we will see what happens when the two slices compete for CPU.

Let's say that we have two processes, the sshd service and a guaranteed pod. Both processes have access to all 4 CPUs and they're trying to use the 100% of the 4 CPUs.

To calculate the percentage of CPU allocated to each process, we can use the following formulas:

* Pod Process: (cpu.weight of pod / total cpu.weight) * number of CPUs
* Ssh Process: (cpu.weight of ssh / total cpu.weight) * number of CPUs

In this case, the total cpu.weight is 237 (137 from kubepods.slice + 100 from system.slice), so:

* Pod Process: (137 / 237) * 4 = 2.31 CPUs or ~231%
* Ssh Process: (100 / 237) * 4 = 1.68 CPUs or ~168%

So pod process would get around 231% of the available CPU (400% -> 4 Cores x 100) and ssh process would get around 168% of the available CPU.

{{<danger>}}
Keep in mind that these calculations are not 100% accurate, since the CFS will try to assign CPU in the fairest way possible and results may vary depending on the system load and other process running on the system.
{{</danger>}}

### Cgroupsv2 configuration for a Pod

In the previous sections we have focused on the configuration at the node level, but let's see what happens when we create a pod on the different QoS.

#### Cgroup configuration for a BestEffort Pod

We will be using this pod definition:

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: cputest
  name: cputest-besteffort
spec:
  containers:
  - image: quay.io/mavazque/trbsht:latest
    name: cputest
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
~~~

Once created, in the node where the pod gets scheduled we can find the cgroup that was created by using these commands:

1. Get container id:

    ~~~sh
    crictl ps | grep cputest-besteffort
    ~~~

    ~~~console
    2be6af51555a1       b67fff43d1e61       4 minutes ago       Running             cputest                     0                   cbce891122629       cputest-besteffort
    ~~~

2. Get the cgroups path:

    ~~~sh
    crictl inspect 2be6af51555a1 | jq '.info.runtimeSpec.linux.cgroupsPath'
    ~~~

    ~~~console
    "kubepods-besteffort-pod7589d90f_83af_4a05_a4ee_8bb078db72b8.slice:cri-containerd:2be6af51555a1d9ebb8678f3254e81b5f3547dfc230b07a2c1067f5d430b7221"
    ~~~

3. With above information, the full path will be `/sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod7589d90f_83af_4a05_a4ee_8bb078db72b8.slice`

If we check the _cpu.max_, _cpu.weight_ and _memory.max_ configuration, this is what we see:

* _cpu.max_ is set to `max 100000`.
* _cpu.weight_ is set to `1`.
* _memory.max_ is set to `max`.

As we can see, the pod is allowed to use as much CPU as it wants, but it has the lowest weight possible which means that it only will get CPU when other processes with higher weight yield some. You can expect a lot of throttling for these pods when the system is under load. On the memory side, it can use as much memory as it wants, but if the cluster requires evicting this pod to reclaim memory in order to schedule more priority pods the container will be OOMKilled. The `max` from the `cpu.max` config means that the processes can use all the CPU time available on the system (which varies depending on the speed of your CPU).

#### Cgroup configuration for a Burstable Pod

We will be using this pod definition:

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: cputest
  name: cputest-burstable
spec:
  containers:
  - image: quay.io/mavazque/trbsht:latest
    name: cputest
    resources:
      requests:
        cpu: 2
        memory: 100Mi
  dnsPolicy: ClusterFirst
  restartPolicy: Always
~~~

Once created, in the node where go the cgroup configuration by following the steps described previously and this is the configuration we see:

* _cpu.max_ is set to `max 100000`.
* _cpu.weight_ is set to `79`.
* _memory.max_ is set to `max`.

The pod will be allowed to use as much CPU as it wants, and the weight has been set to it has certain priority over other processes running on the system. On the memory side it can use as much memory as it wants, but if the cluster requires evicting this pod to reclaim memory in order to schedule more priority pods the container will be OOMKilled. The _cpu.weight_ value `79` comes from the formula we saw earlier (`(((cpuShares - 2) * 9999) / 262142) + 1`):

~~~math
cpuShares = 2 Cores * 1024 = 2048
cpu.weight = (((2048 - 2) * 9999) / 262142) + 1 = 79,04
~~~

#### Cgroup configuration for a Guaranteed Pod

We will be using this pod definition:

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: cputest
  name: cputest-guaranteed
spec:
  containers:
  - image: quay.io/mavazque/trbsht:latest
    name: cputest
    resources:
      requests:
        cpu: 2
        memory: 100Mi
      limits:
        cpu: 2
        memory: 100Mi
  dnsPolicy: ClusterFirst
  restartPolicy: Always
~~~

Once created, in the node where go the cgroup configuration by following the steps described previously and this is the configuration we see:

* _cpu.max_ is set to `200000 100000`.
* _cpu.weight_ is set to `79`.
* _memory.max_ is set to `104857600` (100Mi = 104857600 bytes).

The _cpu.max_ value is different to what we have seen so far, the first value `200000` is the allowed time quota in microseconds for which the process can run during one period. The second value `100000` specific the length of the period. Once the processes consume the time specified by this quota, they will be throttled for the remained of the period and won't be allowed to run until the next period. This specific configuration allows our processes to run every 0.2 seconds of every 1 second (1/5th). On the memory side, the container can use up to 100Mi once it reaches this value if kernel will try to reclaim some memory, if it cannot be reclaimed the container will be OOMKilled.

Even if guaranteed QoS will _ensure_ that your application gets the CPU it wants, sometimes your application may benefit from burstable capabilities since the CPU won't be throttled during peaks (e.g: more visits to a web server).

### How Kubepods Cgroups compete for resources

In the previous examples we have seen how the different pods get different CPU configurations. But what happens if they compete against them for resources? 

In order for the `guaranteed` pods to have more priority than `burstable` pods, and these to have more priority than `besteffort` different weights get set for the three slices. In a 4 CPU node these are the settings we get:

* Guaranteed pods will run under `kubepods.slice` which has a `cpu.weight` of `137`.
* Burstable pods will run under `kubepods.slice/kubepods-burstable.slice` which has a `cpu.weight` of `86`.
* BestEffort pods will run under `kubepods.slice/kubepods-besteffort.slice` which has a `cpu.weight` of `1`.

As we can see from above configuration, the weights define the CPU priority. Keep in mind that pods running inside the same parent slice can compete for resources. In this situation, when they're competing for resources the `total cpu.weight` will be the one from summing all their parent cgroup cpu weights. For example:

We have two burstable pods, these are the cpu weights that will be configured (based on the formulas we have seen so far):

* `bustable1` requests 2 CPUs and gets a _cpu.weight_ of `79`
* `burstable2` requests 1 CPU and gets a _cpu.weight_ of `39`

So this is the CPU each one will get (formula: `(cpu.weight of pod / total cpu.weight) * number of CPUs`):

{{<danger>}}
Keep in mind that these calculations are not 100% accurate, since the CFS will try to assign CPU in the fairest way possible and results may vary depending on the system load and other process running on the system. This calculations assume that there are no guaranteed pods demanding CPU. `223` value comes from summing all the parents cpu weights `137` from `kubepods.slice` and `86` from `kubepods-burstable.slice`.
{{</danger>}}

* `burstable1`: (79/223) * 4 = 1.41 CPU or ~141%
* `burstable2`: (39/223) * 4 = 0.69 CPU or ~69%~

## Closing Thoughts

Even if knowing the low-level details about resource management on Kubernetes may not be needed in a day-to-day basis, it's great knowing how the different pieces are tied together. If you're working on environments were performance and latencies are critical, like in telco environments, knowing this information can make the difference!

On top of that, some of the new features that cgroupsv2 enable are:

* [Container aware OOMKilled](https://www.scrivano.org/posts/2020-08-14-oom-group/): Useful when you have sidecars, this could be used to OOMKill the sidecar container rather than your application container.
* [Running Kubernetes System components root-less](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-in-userns/): More secure Kubernetes environments.
* [Kubernetes Memory QoS](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/): Better overall control of the memory used by pods.

The Kubernetes Memory QoS kind of relates to this post, so I'll be writing a new post covering that in the future.

Finally, in the next section I'll put interesting resources around the topic, some of them were my sources when learning all this stuff.

## Useful Resources

* KubeCon NA 2022 - Cgroupv2 is coming soon to a cluster near you talk. [Slides](https://static.sched.com/hosted_files/kccncna2022/69/cgroupv2-is-coming-soon-to-a-cluster-near-you-kubecon-na-2022.pdf) and [Recording](https://www.youtube.com/watch?v=sgyFCp1CRhA).
* Lisa 2021 - 5 years of cgroup v2 talk. [Slides](https://www.usenix.org/system/files/lisa21_slides_down.pdf)
* cgroups [man page](https://man7.org/linux/man-pages/man7/cgroups.7.html) and kernel [docs](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html).
* [RHEL8 cgroupv2 docs](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/using-cgroups-v2-to-control-distribution-of-cpu-time-for-applications_managing-monitoring-and-updating-the-kernel).
* Martin Heinz [blog on kubernetes cgroups](https://martinheinz.dev/blog/91).
* Kubernetes cgroups [docs](https://kubernetes.io/docs/concepts/architecture/cgroups/).
* Kubernetes manage resources for containers [docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).
* Kubernetes reserve compute resources [docs](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/).