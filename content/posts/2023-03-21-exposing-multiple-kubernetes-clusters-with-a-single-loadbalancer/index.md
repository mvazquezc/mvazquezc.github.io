---
title:  "Exposing multiple Kubernetes clusters with a single load balancer and a single public IP"
author: "Mario"
tags: [ "kubernetes", "openshift", "haproxy", "loadbalancer" ]
url: "/exposing-multiple-kubernetes-clusters-single-lb-and-ip"
draft: false
date: 2023-03-21
#lastmod: 2023-01-19
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Exposing multiple Kubernetes clusters with a single load balancer and a single public IP

My colleague [Alberto Losada](https://twitter.com/_ZaNN_) and I have been working on a lab lately. The lab is composed of three [OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift) clusters on VMs, these VMs are deployed on an isolated libvirt network, which means that we cannot access them from outside the hypervisor.

In order to solve this issue, we wanted to expose the three clusters using the public IP available in the hypervisor. This setup should be valid for any Kubernetes cluster.

## API and Ingress endpoints

In our case, our OpenShift clusters have two different endpoints, one for the Kubernetes API Server, and another one for the cluster Ingress:

* API Endpoint: **6443** port on control plane nodes
* Cluster Ingress: **80** and **443** ports on compute nodes

We have three clusters:

* hub-cluster.linuxera.lab
* managed1.linuxera.lab
* managed2. linuxera.lab

Each cluster has two DNS records:

* api.\<clustername>.\<basedomain> &nbsp;&nbsp;&nbsp;-> points to the hypervisor public IP
* *.apps.\<clustername>.\<basedomain> -> points to the hypervisor public IP

For example, for the `hub-cluster` we will have: `api.hub-cluster.linuxera.lab` and `*.apps.hub-cluster.linuxera.lab`.

## Load balancing requirements

Usually, when you want to expose multiple clusters under the same load balancer you either use different public IPs, or different ports. For example, if you wanted to expose three clusters you could use three different IPs. Your load balancer will listen for connections on these IPs and depending on the IP receiving the request the load balancer will redirect traffic to the cluster exposed by that IP.

A different approach would be using the same IP for the load balancer but different ports, for example cluster1 API will be published in port 6443, cluster2 in port 6444, etc.

In this lab environment we only had a public IP, and we didn't want to use different ports. We wanted to be able to route request based on the destination cluster. On top of that, we didn't want to do TLS termination in the load balancer, instead we wanted the different clusters to do that.

## Detecting destination for the connections

Users using our clusters will connect to the different clusters APIs and ingress endpoints. When they do that, depending on the destination cluster these connections will have different information that we can use to redirect those connections to the proper cluster.

For connections going to the HTTP Ingress endpoint we will use the `Host` header. For TLS connections going to the HTTPS/API ingress endpoints we will use the `SNI` from the TLS handshake.

## Our solution

We decided to go with HAProxy. The architecture looks like this:

![Load Balancer Arch](./lb-arch.png)

And our configuration file:

~~~console
global
    log         127.0.0.1 local2
    maxconn     4000
    daemon

defaults
    mode                    tcp
    log                     global
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen stats-50000
    bind :50000
    mode            http
    log             global
    maxconn 10
    timeout client  100s
    timeout server  100s
    timeout connect 100s
    stats enable
    stats hide-version
    stats refresh 30s
    stats show-node
    stats auth admin:password
    stats uri  /haproxy?stats

frontend apis-6443
    bind :6443
    mode tcp
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }
    acl ACL_hub req_ssl_sni -i api.hub-cluster.linuxera.lab
    acl ACL_managed1 req_ssl_sni -i api.managed1.linuxera.lab
    acl ACL_managed2 req_ssl_sni -i api.managed2.linuxera.lab
    use_backend be_api_hub_6443 if ACL_hub
    use_backend be_api_managed1_6443 if ACL_managed1
    use_backend be_api_managed2_6443 if ACL_managed2

frontend routers-http-80
    bind :80
    mode http
    acl ACL_hub hdr(host) -m reg -i ^[^\.]+\.apps\.hub\.linuxera\.lab
    acl ACL_managed1 hdr(host) -m reg -i ^[^\.]+\.apps\.managed1\.linuxera\.lab
    acl ACL_managed2 hdr(host) -m reg -i ^[^\.]+\.apps\.managed2\.linuxera\.lab
    use_backend be_ingress_hub_80 if ACL_hub
    use_backend be_ingress_managed1_80 if ACL_managed1
    use_backend be_ingress_managed2_80 if ACL_managed2

frontend routers-https-443
    bind :443
    mode tcp
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }
    acl ACL_hub req_ssl_sni -m reg -i ^[^\.]+\.apps\.hub\.linuxera\.lab
    acl ACL_managed1 req_ssl_sni -m reg -i ^[^\.]+\.apps\.managed1\.linuxera\.lab
    acl ACL_managed2 req_ssl_sni -m reg -i ^[^\.]+\.apps\.managed2\.linuxera\.lab
    use_backend be_ingress_hub_443 if ACL_hub
    use_backend be_ingress_managed1_443 if ACL_managed1
    use_backend be_ingress_managed2_443 if ACL_managed2

backend be_api_hub_6443
    mode tcp
    balance source
    option ssl-hello-chk
    server controlplane0 192.168.125.20:6443 check inter 1s
    server controlplane1 192.168.125.21:6443 check inter 1s
    server controlplane2 192.168.125.22:6443 check inter 1s
    
backend be_api_managed1_6443
    mode tcp
    balance source
    option ssl-hello-chk
    server controlplane0 192.168.125.30:6443 check inter 1s
    server controlplane1 192.168.125.31:6443 check inter 1s
    server controlplane2 192.168.125.32:6443 check inter 1s

backend be_api_managed2_6443
    mode tcp
    balance source
    option ssl-hello-chk
    server controlplane0 192.168.125.40:6443 check inter 1s
    server controlplane1 192.168.125.41:6443 check inter 1s
    server controlplane2 192.168.125.42:6443 check inter 1s

backend be_ingress_hub_80
    mode http
    balance hdr(Host)
    hash-type consistent
    option forwardfor
    http-send-name-header Host
    server compute0 192.168.125.23:80 check inter 1s
    server compute1 192.168.125.24:80 check inter 1s
    server compute2 192.168.125.25:80 check inter 1s

backend be_ingress_hub_443
    mode tcp
    balance source
    option ssl-hello-chk
    server compute0 192.168.125.23:443 check inter 1s
    server compute1 192.168.125.24:443 check inter 1s
    server compute2 192.168.125.25:443 check inter 1s

backend be_ingress_managed1_80
    mode http
    balance hdr(Host)
    hash-type consistent
    option forwardfor
    http-send-name-header Host
    server compute0 192.168.125.33:80 check inter 1s
    server compute1 192.168.125.34:80 check inter 1s
    server compute2 192.168.125.35:80 check inter 1s

backend be_ingress_managed1_443
    mode tcp
    balance source
    option ssl-hello-chk
    server compute0 192.168.125.33:443 check inter 1s
    server compute1 192.168.125.34:443 check inter 1s
    server compute2 192.168.125.35:443 check inter 1s

backend be_ingress_managed2_80
    mode http
    balance hdr(Host)
    hash-type consistent
    option forwardfor
    http-send-name-header Host
    server compute0 192.168.125.43:80 check inter 1s
    server compute1 192.168.125.44:80 check inter 1s
    server compute2 192.168.125.45:80 check inter 1s

backend be_ingress_managed2_443
    mode tcp
    balance source
    option ssl-hello-chk
    server compute0 192.168.125.43:443 check inter 1s
    server compute1 192.168.125.44:443 check inter 1s
    server compute2 192.168.125.45:443 check inter 1s
~~~

From this configuration file we can remark the following parameters:

1. Redirect to different API backends based on SNI

    ~~~console
    acl ACL_hub req_ssl_sni -i api.hub-cluster.linuxera.lab
    acl ACL_managed1 req_ssl_sni -i api.managed1.linuxera.lab
    acl ACL_managed2 req_ssl_sni -i api.managed2.linuxera.lab
    ~~~

2. Redirect to different http ingress backends based on Host header and wildcard domain

    ~~~console
    acl ACL_hub hdr(host) -m reg -i ^[^\.]+\.apps\.hub\.linuxera\.lab
    acl ACL_managed1 hdr(host) -m reg -i ^[^\.]+\.apps\.managed1\.linuxera\.lab
    acl ACL_managed2 hdr(host) -m reg -i ^[^\.]+\.apps\.managed2\.linuxera\.lab
    ~~~

3. Redirect to different https ingress backends based on SNI and wildcard domain

    {{<attention>}}
We also add `tcp-request inspect-delay 5s` for HAProxy to have enough time to inspect the connection.
    {{</attention>}}

    ~~~console
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }
    acl ACL_hub req_ssl_sni -m reg -i ^[^\.]+\.apps\.hub\.linuxera\.lab
    acl ACL_managed1 req_ssl_sni -m reg -i ^[^\.]+\.apps\.managed1\.linuxera\.lab
    acl ACL_managed2 req_ssl_sni -m reg -i ^[^\.]+\.apps\.managed2\.linuxera\.lab
    ~~~