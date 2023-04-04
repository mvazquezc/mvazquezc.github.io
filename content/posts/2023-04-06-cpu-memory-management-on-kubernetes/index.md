---
title:  "CPU and Memory Management on Kubernetes"
author: "Mario"
tags: [ "kubernetes", "openshift", "cgroups" ]
url: "/cpu-memory-management-kubernetes"
draft: true
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

# CPU and Memory Management on Kubernetes

In this post I'll try to explain how CPU and Memory management works under the hood on Kubernetes. If you ever wondered what happens when you set `requests` and `limits` for your pods, keep reading!

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

7. In cgroupvs1 there was `cpu shares` concept, in cgroupsv2 this changed and now we use `cpu weights`. All weights are in the range [1, 10000] with the default at 100. This allows symmetric multiplicative biases in both directions at fine enough granularity while staying in the intuitive range. If we wanted to give `appA` a `30%` of the CPU and `appB` the other `70%` providing that the parent cgroup CPU weight is set to 100 this is the configuration we will apply:

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

Now we should have a clear understanding on how the basics work, next section will introduce these concepts applied to Kubernetes.



## Useful Resources

* https://www.usenix.org/system/files/lisa21_slides_down.pdf
* https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/using-cgroups-v2-to-control-distribution-of-cpu-time-for-applications_managing-monitoring-and-updating-the-kernel
* https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html