---
title:  "Containers under the Hood"
author: "Mario"
tags: [ "containers", "linux", "namespaces", "nsenter", "unshare", "kernel", "selinux" ]
url: "/containers-under-the-hood/"
draft: false
date: 2021-03-25
#lastmod: 2021-03-25
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Containers are Linux

You probably already heard this expression, in today's post we are going to desmitify container technologies by decomposing them part by part and describing which Linux technologies make containers possible.

We can describe a container as an isolated process running on a host. In order to isolate the process the container runtimes leverage Linux kernel technologies such as: namespaces, chroots, cgroups, etc. plus security layers like SELinux.

We will see how we can leverage these technologies on Linux in order to build and run our own containers.

## Container File Systems (a.k.a rootfs)

Whenever you pull an image container from a container registry, you're downloading just a tarball. We can say container images are just tarballs.

There are multiple ways to get a rootfs that we can use in order to run our containers, for this blogpost we're going to download an already built rootfs for Alpine Linux.

There are tools such as [buildroot](https://buildroot.org/) that make it really easy to create our own rootfs. We will see how to create our own rootfs using buildroot on a future post.

As earlier mentioned, let's download the x86_64 rootfs for [Alpine Linux](https://alpinelinux.org/downloads/):

~~~sh
mkdir /var/tmp/alpine-rootfs/ && cd $_
curl https://dl-cdn.alpinelinux.org/alpine/v3.12/releases/x86_64/alpine-minirootfs-3.12.3-x86_64.tar.gz -o rootfs.tar.gz
~~~

We can extract the rootfs on the temporary folder we just created:

~~~sh
tar xfz rootfs.tar.gz && rm -f rootfs.tar.gz
~~~

If we take a look at the extracted files:

~~~sh
tree -L 1
~~~

As you can see, the result looks like a Linux system. We have some well known directories in the Linux Filesystem Hierarchy Standard such as: `bin`, `tmp`, `dev`, `opt`, etc.

~~~
.
├── bin
├── dev
├── etc
├── home
├── lib
├── media
├── mnt
├── opt
├── proc
├── root
├── run
├── sbin
├── srv
├── sys
├── tmp
├── usr
└── var
~~~

### chroot

Chroot is an operation that changes the root directory for the current running process and their children. A process that runs inside a chroot cannot access files and commands outside the chrooted directory tree. 

That being said, we can now chroot into the rootfs environment we extracted in the previous step and run a shell to poke around:

1. Create the chroot jail

    ~~~sh
    sudo chroot /var/tmp/alpine-rootfs/ /bin/sh
    ~~~
2. Check the os-release

    ~~~sh
    cat /etc/os-release
    ~~~

    ~~~
    NAME="Alpine Linux"
    ID=alpine
    VERSION_ID=3.12.3
    PRETTY_NAME="Alpine Linux v3.12"
    HOME_URL="https://alpinelinux.org/"
    BUG_REPORT_URL="https://bugs.alpinelinux.org/"
    ~~~
3. Try to list /tmp/alpine-rootfs folder

    ~~~sh
    ls /var/tmp/alpine-rootfs
    ~~~

    ~~~
    ls: /var/tmp/alpine-rootfs: No such file or directory
    ~~~

As you can see we only have visibility of the contents of the rootfs we've chroot into.

We can now install python and run a simple http server for example:

1. Install python3

    ~~~sh
    apk add python3
    ~~~
2. Run a simple http server

    > **NOTE**: When we execute the Python interpreter we're actually running it from `/var/tmp/alpine-rootfs/usr/bin/python3`

    ~~~sh
    python3 -m http.server
    ~~~
3. If you open a new shell on your system (even if it's outside of the chroot) you will be able to reach the http server we just created:

    ~~~sh
    curl http://127.0.0.1:8000
    ~~~

## namespaces

At this point we were able to work with a tarball like if it was a different system, but we're not isolating the processed from the host system like containers do.

Let's check the level of isolation:

1. In a shell outside the chroot run a `ping` command:

    ~~~sh
    ping 127.0.0.1
    ~~~
2. Mount the `proc` filesystem inside the chrooted shell

    > **NOTE**: If you're still running the python http server you can kill the process

    ~~~sh
    mount -t proc proc /proc
    ~~~
2. Run a `ps` command inside the chroot and try to find the `ping` command:

    ~~~sh
    ps -ef | grep "ping 127.0.0.1"
    ~~~

    ~~~
    387870 1000      0:00 ping 127.0.0.1
    388204 root      0:00 grep ping 127.0.0.1
    ~~~
3. We have visibility over the host system processes, that's not great. On top of that, our chroot is running as root so we can even kill the process:

    ~~~sh
    pkill -f "ping 127.0.0.1"
    ~~~

Now is when we introduce namespaces. 

**Linux namespaces** are a feature of the Linux kernel that partitions kernel resources so one process will only see a set of resources while a different process can see a different set of resources. 

These resources may exist in multiple spaces. The list of existing namespaces are:

|Namespace|Isolates|
|---------|--------|
| Cgroup | Cgroup root directory |
| IPC | System V IPC, POSIX message queues |
| Network | Network devices, stacks, prots, etc. |
| Mount | Mount points |
| PID | Process IDs |
| Time | Boot and monotonic clocks |
| User | User and Group IDs |
| UTS | Hostname and NIS domain name |

<br>

### Creating namespaces with unshare

Creating namespaces is just a single [syscall (unshare)](https://man7.org/linux/man-pages/man2/unshare.2.html). There is also a `unshare` command line tool that provides a nice wrapper around the syscall.

We are going to use the `unshare` command line to create namespaces manually. Below example will create a `PID` namespace for the chrooted shell:

1. Exit the chroot we have already running

    > **NOTE**: Run below command on the chrooted shell

    ~~~sh
    exit
    ~~~
2. Create the `PID` namespace and run the chrooted shell inside the namespace

    ~~~sh
    sudo unshare -p -f --mount-proc=/var/tmp/alpine-rootfs/proc chroot /var/tmp/alpine-rootfs/ /bin/sh
    ~~~
3. Now that we have created our new process namespace, we will see that our shell thinks its PID is 1:

    ~~~sh
    ps -ef
    ~~~

    > **NOTE**: As you can see, we no longer see the host system processes
    ~~~
    PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    2 root      0:00 ps -ef
    ~~~
4. Since we didn't create a namespace for the network we can still see the whole network stack from the host system:

    ~~~sh
    ip -o a
    ~~~

    > **NOTE**: Below output might vary on your system

    ~~~
    1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
    1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
    4: wlp82s0    inet 192.168.0.160/24 brd 192.168.0.255 scope global dynamic wlp82s0\       valid_lft 6555sec preferred_lft 6555sec
    4: wlp82s0    inet6 fe80::4e03:6176:40f0:b862/64 scope link \       valid_lft forever preferred_lft forever
    ~~~

### Entering namespaces with nsenter

One powerful thing about namespaces is that they're pretty flexible, for example you can have processes with some separated namespaces and some shared namespaces. One example in the Kubernetes world will be containers running in pods: Containers will have different `PID` namespaces but they will share the `Network` namespace.

There is a [syscall (setns)](https://man7.org/linux/man-pages/man2/setns.2.html) that can be used to reassociate a thread with a namespace. The `nsenter` command line tool will help with that.

We can check the namespaces for a given process by querying the `/proc` filesystem:

1. From a shell outside the chroot get the PID for the chrooted shell

    ~~~sh
    UNSHARE_PPID=$(ps -ef | grep "sudo unshare" | grep chroot | awk '{print $2}')
    UNSHARE_PID=$(ps -ef | grep ${UNSHARE_PPID} | grep chroot | grep -v sudo | awk '{print $2}')
    SHELL_PID=$(ps -ef | grep ${UNSHARE_PID} | grep -v chroot |  grep /bin/sh | awk '{print $2}')
    ps -ef | grep ${UNSHARE_PID} | grep -v chroot |  grep /bin/sh
    ~~~

    ~~~
    root      390072  390071  0 12:32 pts/1    00:00:00 /bin/sh
    ~~~
2. From a shell outside the chroot get the namespaces for the shell process:

    ~~~sh
    sudo ls -l /proc/${SHELL_PID}/ns
    ~~~

    ~~~
    total 0
    lrwxrwxrwx. 1 root root 0 mar 25 12:54 cgroup -> 'cgroup:[4026531835]'
    lrwxrwxrwx. 1 root root 0 mar 25 12:54 ipc -> 'ipc:[4026531839]'
    lrwxrwxrwx. 1 root root 0 mar 25 12:54 mnt -> 'mnt:[4026532266]'
    lrwxrwxrwx. 1 root root 0 mar 25 12:54 net -> 'net:[4026532008]'
    lrwxrwxrwx. 1 root root 0 mar 25 12:54 pid -> 'pid:[4026532489]'
    lrwxrwxrwx. 1 root root 0 mar 25 12:54 pid_for_children -> 'pid:[4026532489]'
    lrwxrwxrwx. 1 root root 0 mar 25 12:54 time -> 'time:[4026531834]'
    lrwxrwxrwx. 1 root root 0 mar 25 12:54 time_for_children -> 'time:[4026531834]'
    lrwxrwxrwx. 1 root root 0 mar 25 12:54 user -> 'user:[4026531837]'
    lrwxrwxrwx. 1 root root 0 mar 25 12:54 uts -> 'uts:[4026531838]'

    ~~~
3. Earlier we saw how we were just setting a different `PID` namespace, let's see the difference between the `PID` namespace configured for our chroot shell and for the regular shell:

    > **NOTE**: Below commands must be run from a shell outside the chroot:
    
    1. Get `PID` namespace for the chrooted shell:
    
        ~~~sh
        sudo ls -l /proc/${SHELL_PID}/ns/pid
        ~~~

        ~~~
        lrwxrwxrwx. 1 root root 0 mar 25 12:54 pid -> pid:[4026532489]
        ~~~
    2. Get `PID` namespace for the regular shell:

        ~~~sh
        sudo ls -l /proc/$$/ns/pid
        ~~~

        ~~~
        lrwxrwxrwx. 1 mario mario 0 mar 25 12:55 pid -> pid:[4026531836]
        ~~~
    3. As you can see, both processes are using a different `PID` namespace. We saw that network stack was still visible, let's see if there is any difference in the `Network` namespace for both processes. Let's start with the chrooted shell:

        ~~~sh
        sudo ls -l /proc/${SHELL_PID}/ns/net
        ~~~

        ~~~
        lrwxrwxrwx. 1 root root 0 mar 25 12:54 net -> net:[4026532008]
        ~~~
    4. Now, time to get the one for the regular shell:

        ~~~sh
        sudo ls -l /proc/$$/ns/net
        ~~~

        ~~~
        lrwxrwxrwx. 1 mario mario 0 mar 25 12:55 net -> net:[4026532008]
        ~~~
    5. As you can see from above outputs, both processes are using the same `Network` namespace.

If we want to join a process to an existing namespace we can do that using `nsenter` as we said before, let's do that.

1. Open a new shell outside the chroot
2. We want run a new chrooted shell and join the already existing `PID` namespace we created earlier:

    ~~~sh
    # Get the previous unshare PPID
    UNSHARE_PPID=$(ps -ef | grep "sudo unshare" | grep chroot | awk '{print $2}')
    # Get the previous unshare PID
    UNSHARE_PID=$(ps -ef | grep ${UNSHARE_PPID} | grep chroot | grep -v sudo | awk '{print $2}')
    # Get the previous chrooted shell PID
    SHELL_PID=$(ps -ef | grep ${UNSHARE_PID} | grep -v chroot |  grep /bin/sh | awk '{print $2}')
    # We will enter the previous PID namespace, remount the /proc filesystem and run a new chrooted shell
    sudo nsenter --pid=/proc/${SHELL_PID}/ns/pid unshare -f --mount-proc=/tmp/alpine-rootfs/proc chroot /tmp/alpine-rootfs/ /bin/sh
    ~~~
3. From the new chrooted shell we can run a `ps` command and we should see the existing processes from the previous chrooted shell:

    ~~~sh
    ps -ef
    ~~~

    ~~~
    PID   USER     TIME  COMMAND
      1   root     0:00  /bin/sh
      4   root     0:00  unshare -f --mount-proc=/tmp/alpine-rootfs/proc chroot /tmp/alpine-rootfs/ /bin/sh
      5   root     0:00  /bin/sh
      6   root    0:00  ps -ef
    ~~~
4. We have entered the already existing `PID` namespace used by our previous chrooted shell and we can see that running a `ps` command from the new shell (PID 5) we can see the first shell (PID 1).

## Injecting files or directories into the chroot

Containers are usually inmutables, that means that we cannot create or edit directories or files into the chroot. Sometimes we will need to inject files or directories either for storage or configuration. We are going to show how we can create some files on the host system and expose them as read-only to the chrooted shell using `mount`.

1. Create a folder in the host system to host some read-only config files:

    ~~~sh
    sudo mkdir -p /var/tmp/alpine-container-configs/
    echo "Test" | sudo tee -a /var/tmp/alpine-container-configs/app-config
    echo "Test2" | sudo tee -a /var/tmp/alpine-container-configs/srv-config
    ~~~
2. Create a folder in the rootfs directory to use it as mount point:

    ~~~sh
    sudo mkdir -p /var/tmp/alpine-rootfs/etc/myconfigs
    ~~~
3. Run a bind mount:

    ~~~sh
    sudo mount --bind -o ro /var/tmp/alpine-container-configs /var/tmp/alpine-rootfs/etc/myconfigs
    ~~~
4. Run a chrooted shell and check the mounted files:

    > **NOTE**: You can exit from the already existing chrooted shells before creating this one
    
    ~~~sh
    sudo unshare -p -f --mount-proc=/var/tmp/alpine-rootfs/proc chroot /var/tmp/alpine-rootfs/ /bin/sh
    ~~~

    ~~~sh
    ls -l /etc/myconfigs/
    ~~~

    ~~~
    total 8
    -rw-r--r--    1 root     root             5 Mar 25 13:28 app-config
    -rw-r--r--    1 root     root             6 Mar 25 13:28 srv-config
    ~~~
5. If we try to edit the files from the chrooted shell, this is what happens:

    ~~~sh
    echo "test3" >> /etc/myconfigs/app-config
    ~~~

    > **NOTE**: We cannot edit/create files since the mount is read-only
    ~~~
    /bin/sh: can't create /etc/myconfigs/app-config: Read-only file system
    ~~~
6. If we want to unmount the files we can run the command below from the host system:

    ~~~sh
    sudo umount /var/tmp/alpine-rootfs/etc/myconfigs
    ~~~

## CGroups

Control groups allow the kernel to restrict resources like memory and CPU for specific processes. In this case we are going to create a new CGroup for our chrooted shell so it cannot use more than 200MB of RAM.

Kernel exposes `cgroups` at the `/sys/fs/cgroup` directory:

~~~sh
ls /sys/fs/cgroup/
~~~

~~~
cgroup.controllers      cgroup.stat             cpuset.cpus.effective  io.cost.model  machine.slice     system.slice
cgroup.max.depth        cgroup.subtree_control  cpuset.mems.effective  io.cost.qos    memory.numa_stat  user.slice
cgroup.max.descendants  cgroup.threads          cpu.stat               io.pressure    memory.pressure
cgroup.procs            cpu.pressure            init.scope             io.stat        memory.stat
~~~

1. Let's create a new cgroup, we just need to create a folder for that to happen:

    ~~~sh
    sudo mkdir /sys/fs/cgroup/alpinecgroup
    ~~~

    ~~~
    ls /sys/fs/cgroup/alpinecgroup/
    ~~~

    > **NOTE**: The kernel automatically populated the folder
    ~~~
    cgroup.controllers      cgroup.stat             io.pressure          memory.max        memory.swap.current  pids.max
    cgroup.events           cgroup.subtree_control  memory.current       memory.min        memory.swap.events
    cgroup.freeze           cgroup.threads          memory.events        memory.numa_stat  memory.swap.high
    cgroup.max.depth        cgroup.type             memory.events.local  memory.oom.group  memory.swap.max
    cgroup.max.descendants  cpu.pressure            memory.high          memory.pressure   pids.current
    cgroup.procs            cpu.stat                memory.low           memory.stat       pids.events
    ~~~
2. Now, we just need to adjust the memory value by modifying the required files:

    ~~~sh
    # Set a limit of 200MB of RAM
    echo "200000000" | sudo tee -a /sys/fs/cgroup/alpinecgroup/memory.max
    # Disable swap
    echo "0" | sudo tee -a /sys/fs/cgroup/alpinecgroup/memory.swap.max
    ~~~
3. Finally, we need to assign this CGroup to our chrooted shell:

    ~~~sh
    # Get the previous unshare PPID
    UNSHARE_PPID=$(ps -ef | grep "sudo unshare" | grep chroot | awk '{print $2}')
    # Get the previous unshare PID
    UNSHARE_PID=$(ps -ef | grep ${UNSHARE_PPID} | grep chroot | grep -v sudo | awk '{print $2}')
    # Get the previous chrooted shell PID
    SHELL_PID=$(ps -ef | grep ${UNSHARE_PID} | grep -v chroot |  grep /bin/sh | awk '{print $2}')
    # Assign the shell process to the cgroup
    echo ${SHELL_PID} | sudo tee -a /sys/fs/cgroup/alpinecgroup/cgroup.procs
    ~~~
4. In order to test the cgroup we will create a dumb python script in the chrooted shell:

    ~~~sh
    # Mount the /dev fs since we need to read data from urandom
    mount -t devtmpfs dev /dev
    # Create the python script
    cat <<EOF > /opt/dumb.py
    f = open("/dev/urandom", "r", encoding = "ISO-8859-1")
    data = ""
    i=0
    while i < 20:
        data += f.read(10000000) # 10mb
        i += 1
        print("Used %d MB" % (i * 10))
    EOF
    ~~~
5. Run the script:

    ~~~sh
    python3 /opt/dumb.py
    ~~~

    > **NOTE**: The process was killed before it reached the memory limit.
    ~~~
    python3 /opt/dumb.py
    Used 10 MB
    Used 20 MB
    Used 30 MB
    Used 40 MB
    Used 50 MB
    Used 60 MB
    Used 70 MB
    Used 80 MB
    Used 90 MB
    Used 100 MB
    Used 110 MB
    Used 120 MB
    Used 130 MB
    Used 140 MB
    Used 150 MB
    Used 160 MB
    Used 170 MB
    Killed
    ~~~
6. We can now close the chrooted shell and remove the cgroup

    1. Exit the chrooted shell:
    
        ~~~sh
        exit
        ~~~
    
    > **NOTE**: A CGroup cannot be removed until all the attached processes are finished.

    2. Remove the cgroup:

        ~~~sh
        sudo rmdir /sys/fs/cgroup/alpinecgroup/
        ~~~

## Container security and capabilities

As you know, Linux containers run directly on top of the host system and share multiple resources like the Kernel, filesystems, network stack, etc. If an attacker breaks out of the container confinement security risks will arise. 

In order to limit the attack surface there are many technologies involved in limiting the power of processes running in the container such as SELinux, Security Compute Profiles and Linux Capabilities.

You can learn more in [this blogpost](https://linuxera.org/container-security-capabilities-seccomp/).

# Closing Thoughts

Containers are not new, they use technologies that have been present in the Linux kernel for a long time. Tools like [Podman](https://podman.io/) or [Docker](https://www.docker.com/) make running containers easy for everyone by abstracting the different Linux technologies used under the hood from the user.

I hope that now you have a better understanding of what technologies are used when you run containers on your systems.

# Sources

* [Containers from from Scratch](https://ericchiang.github.io/post/containers-from-scratch/)
* [Creating CGroupsv2](https://facebookmicrosites.github.io/cgroup2/docs/create-cgroups.html)
