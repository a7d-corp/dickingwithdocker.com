---
title: "Forcing Kubernetes to use a secondary interface"
date: "2016-11-20"
categories: 
  - "containers"
  - "kubernetes"
  - "orchestration"
tags: 
  - "kubernetes"
  - "orchestration"
---

Following on from my [previous post](http://dickingwithdocker.com/deploying-kubernetes-1-4-on-ubuntu-xenial-with-kubeadm/), I discovered rather to my dismay that although I had my nodes initially communicating over the secondary interface, the weave services (and thus my inter-pod traffic) was all going over the public interface.

As these are VPSes, they have a public IP on eth0 and a VLAN IP on eth1, so it makes sense for all inter-pod traffic to stay internal. If I check the logs for one of the weave-net containers, we can see all comms are going via the 1.1.1.x IPs (for the purposes of this post they are the public IPs of each VPS):

```
root@kube1:~# kubectl logs weave-net-y9k33 -c weave --namespace=kube-system
INFO: 2016/11/16 21:04:59.148121 Command line options: map[docker-api: datapath:datapath http-addr:127.0.0.1:6784 ipalloc-init:consensus=3 nickname:kube3.domain.com no-dns:true status-addr:0.0.0.0:6782 ipalloc-range:10.32.0.0/12 name:ea:93:cb:06:2f:a8 port:6783]
INFO: 2016/11/16 21:04:59.151781 Communication between peers is unencrypted.
INFO: 2016/11/16 21:04:59.159386 Our name is ea:93:cb:06:2f:a8(kube3.domain.com)
INFO: 2016/11/16 21:04:59.159480 Launch detected - using supplied peer list: [1.1.1.93 1.1.1.94 1.1.1.239]
INFO: 2016/11/16 21:04:59.159543 [allocator ea:93:cb:06:2f:a8] No valid persisted data
INFO: 2016/11/16 21:04:59.160613 [allocator ea:93:cb:06:2f:a8] Initialising via deferred consensus
INFO: 2016/11/16 21:04:59.160699 Sniffing traffic on datapath (via ODP)
INFO: 2016/11/16 21:04:59.175904 -&gt;[1.1.1.93:6783] attempting connection
INFO: 2016/11/16 21:04:59.177290 -&gt;[1.1.1.94:6783] attempting connection
INFO: 2016/11/16 21:04:59.177726 -&gt;[1.1.1.239:6783] attempting connection
INFO: 2016/11/16 21:04:59.202281 Listening for HTTP control messages on 127.0.0.1:6784
INFO: 2016/11/16 21:04:59.204496 Listening for metrics requests on 0.0.0.0:6782
INFO: 2016/11/16 21:04:59.217089 -&gt;[1.1.1.239:41223] connection accepted
INFO: 2016/11/16 21:04:59.230002 -&gt;[1.1.1.94:6783|66:06:c1:58:56:30(kube2.domain.com)]: connection ready; using protocol version 2
INFO: 2016/11/16 21:04:59.232908 overlay_switch -&gt;[66:06:c1:58:56:30(kube2.domain.com)] using fastdp
INFO: 2016/11/16 21:04:59.233058 -&gt;[1.1.1.94:6783|66:06:c1:58:56:30(kube2.domain.com)]: connection added (new peer)
INFO: 2016/11/16 21:04:59.235624 EMSGSIZE on send, expecting PMTU update (IP packet was 60028 bytes, payload was 60020 bytes)
INFO: 2016/11/16 21:04:59.235742 overlay_switch -&gt;[66:06:c1:58:56:30(kube2.domain.com)] using sleeve
INFO: 2016/11/16 21:04:59.235766 -&gt;[1.1.1.94:6783|66:06:c1:58:56:30(kube2.domain.com)]: connection fully established
INFO: 2016/11/16 21:04:59.236458 -&gt;[1.1.1.93:6783|be:cc:0f:5b:ff:be(kube1.domain.com)]: connection ready; using protocol version 2
INFO: 2016/11/16 21:04:59.236550 overlay_switch -&gt;[be:cc:0f:5b:ff:be(kube1.domain.com)] using fastdp
INFO: 2016/11/16 21:04:59.236688 -&gt;[1.1.1.93:6783|be:cc:0f:5b:ff:be(kube1.domain.com)]: connection added (new peer)
INFO: 2016/11/16 21:04:59.237856 -&gt;[1.1.1.93:6783|be:cc:0f:5b:ff:be(kube1.domain.com)]: connection fully established
INFO: 2016/11/16 21:04:59.239273 -&gt;[1.1.1.239:41223|ea:93:cb:06:2f:a8(kube3.domain.com)]: connection shutting down due to error: cannot connect to ourself
INFO: 2016/11/16 21:04:59.239670 EMSGSIZE on send, expecting PMTU update (IP packet was 60028 bytes, payload was 60020 bytes)
INFO: 2016/11/16 21:04:59.239848 overlay_switch -&gt;[66:06:c1:58:56:30(kube2.domain.com)] using fastdp
INFO: 2016/11/16 21:04:59.239943 sleeve -&gt;[1.1.1.94:6783|66:06:c1:58:56:30(kube2.domain.com)]: Effective MTU verified at 1438
INFO: 2016/11/16 21:04:59.240347 -&gt;[1.1.1.239:6783|ea:93:cb:06:2f:a8(kube3.domain.com)]: connection shutting down due to error: cannot connect to ourself
INFO: 2016/11/16 21:04:59.241118 sleeve -&gt;[1.1.1.93:6783|be:cc:0f:5b:ff:be(kube1.domain.com)]: Effective MTU verified at 1438
INFO: 2016/11/16 21:05:00.015599 Discovered local MAC ea:93:cb:06:2f:a8
10.36.0.0
```

The reasons for this are not related to Kubeadm, they're in fact rooted in the behaviour of the Kubelet service - by default its name will be the same as the hostname and it's IP address will be that of the default gateway. There are a couple of ways to work around this, depending on what method works for you - neither are perfect but both work well. Firstly, for systemd-based distros you can add a drop-in file to force the hostname the kubelet uses (this does obviously assume that you have a working A record for the hostname you want to use):

```
root@kube1:~# root@kube1:~# echo -e "[Service]\\nEnvironment=\\"KUBELET_EXTRA_ARGS=--hostname-override=kube1.int.domain.com\\"" > /etc/systemd/system/kubelet.service.d/09-hostname-override.conf
root@kube1:~# cat /etc/systemd/system/kubelet.service.d/09-hostname-override.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--hostname-override=kube1.int.domain.com"
```

The alternative to this is to place entries in the hosts file on each node:

```
root@kube2:~# head -n 3 /etc/hosts
172.16.0.128    kube1.domain.com
172.16.0.129    kube2.domain.com
172.16.0.130    kube3.domain.com
```

Run through [the steps](http://dickingwithdocker.com/deploying-kubernetes-1-4-on-ubuntu-xenial-with-kubeadm/) to bootstrap your cluster, then check the cluster is using the VLAN addresses:

```
root@kube1:~# kubectl get pods -o wide --namespace=kube-system
NAME                                          READY     STATUS    RESTARTS   AGE       IP             NODE
dummy-2088944543-phsxr                        1/1       Running   0          3m        172.16.0.128   kube1.domain.com
etcd-kube1.domain.com                         1/1       Running   0          3m        172.16.0.128   kube1.domain.com
kube-apiserver-kube1.domain.com               1/1       Running   0          3m        172.16.0.128   kube1.domain.com
kube-controller-manager-kube1.domain.com      1/1       Running   0          3m        172.16.0.128   kube1.domain.com
kube-discovery-1150918428-0g9sg               1/1       Running   0          3m        172.16.0.128   kube1.domain.com
kube-dns-654381707-uy4sx                      2/3       Running   0          3m        10.40.0.0      kube1.domain.com
kube-proxy-505si                              1/1       Running   0          23s       172.16.0.130   kube3.domain.com
kube-proxy-f6ikp                              1/1       Running   0          23s       172.16.0.128   kube1.domain.com
kube-proxy-kpsr6                              1/1       Running   0          22s       172.16.0.129   kube2.domain.com
kube-scheduler-kube1.domain.com               1/1       Running   0          3m        172.16.0.128   kube1.domain.com
weave-net-c2gjh                               2/2       Running   1          36s       172.16.0.129   kube2.domain.com
weave-net-ics9d                               2/2       Running   1          36s       172.16.0.130   kube3.domain.com
weave-net-thrtu                               2/2       Running   0          36s       172.16.0.128   kube1.domain.com
```

And also the logs from one of the weave-net containers:

```
root@kube1:~# kubectl logs weave-net-xph9z -c weave --namespace=kube-system
INFO: 2016/11/20 20:57:55.855610 Command line options: map[docker-api: datapath:datapath ipalloc-init:consensus=3 ipalloc-range:10.32.0.0/12 nickname:kube1.domain.com no-dns:true status-addr:0.0.0.0:6782 http-addr:127.0.0.1:6784 name:be:cc:0f:5b:ff:be port:6783]
INFO: 2016/11/20 20:57:55.867839 Communication between peers is unencrypted.
INFO: 2016/11/20 20:57:55.871156 Our name is be:cc:0f:5b:ff:be(kube1.domain.com)
INFO: 2016/11/20 20:57:55.871191 Launch detected - using supplied peer list: [172.16.0.128 172.16.0.129 172.16.0.130]
INFO: 2016/11/20 20:57:55.871250 [allocator be:cc:0f:5b:ff:be] No valid persisted data
INFO: 2016/11/20 20:57:55.872404 [allocator be:cc:0f:5b:ff:be] Initialising via deferred consensus
INFO: 2016/11/20 20:57:55.872429 Sniffing traffic on datapath (via ODP)
INFO: 2016/11/20 20:57:55.881339 Listening for HTTP control messages on 127.0.0.1:6784
INFO: 2016/11/20 20:57:55.881763 Listening for metrics requests on 0.0.0.0:6782
INFO: 2016/11/20 20:57:55.882000 ->[172.16.0.129:6783] attempting connection
INFO: 2016/11/20 20:57:55.883211 ->[172.16.0.130:6783] attempting connection
INFO: 2016/11/20 20:57:55.888385 ->[172.16.0.128:6783] attempting connection
INFO: 2016/11/20 20:57:55.902181 ->[172.16.0.130:6783] error during connection attempt: dial tcp4 :0->172.16.0.130:6783: getsockopt: connection refused
INFO: 2016/11/20 20:57:55.902291 ->[172.16.0.129:6783] error during connection attempt: dial tcp4 :0->172.16.0.129:6783: getsockopt: connection refused
INFO: 2016/11/20 20:57:55.902479 ->[172.16.0.128:50824] connection accepted
INFO: 2016/11/20 20:57:55.955552 ->[172.16.0.128:50824|be:cc:0f:5b:ff:be(kube1.domain.com)]: connection shutting down due to error: cannot connect to ourself
INFO: 2016/11/20 20:57:55.956158 ->[172.16.0.128:6783|be:cc:0f:5b:ff:be(kube1.domain.com)]: connection shutting down due to error: cannot connect to ourself
INFO: 2016/11/20 20:57:56.367102 [allocator be:cc:0f:5b:ff:be] Claim 10.32.0.2/12 for weave:expose: is in the range 10.32.0.0/12, but the allocator is not initialized yet; will try later.
INFO: 2016/11/20 20:57:58.172334 ->[172.16.0.130:6783] attempting connection
INFO: 2016/11/20 20:57:58.172916 ->[172.16.0.130:6783] error during connection attempt: dial tcp4 :0->172.16.0.130:6783: getsockopt: connection refused
INFO: 2016/11/20 20:57:58.763326 ->[172.16.0.129:6783] attempting connection
INFO: 2016/11/20 20:57:58.763917 ->[172.16.0.129:6783] error during connection attempt: dial tcp4 :0->172.16.0.129:6783: getsockopt: connection refused
INFO: 2016/11/20 20:58:00.347532 ->[172.16.0.130:6783] attempting connection
INFO: 2016/11/20 20:58:00.348152 ->[172.16.0.130:6783] error during connection attempt: dial tcp4 :0->172.16.0.130:6783: getsockopt: connection refused
INFO: 2016/11/20 20:58:02.619781 ->[172.16.0.129:6783] attempting connection
INFO: 2016/11/20 20:58:02.620873 ->[172.16.0.129:6783] error during connection attempt: dial tcp4 :0->172.16.0.129:6783: getsockopt: connection refused
INFO: 2016/11/20 20:58:05.199335 ->[172.16.0.129:6783] attempting connection
INFO: 2016/11/20 20:58:05.200113 ->[172.16.0.129:6783] error during connection attempt: dial tcp4 :0->172.16.0.129:6783: getsockopt: connection refused
INFO: 2016/11/20 20:58:05.541251 ->[172.16.0.130:6783] attempting connection
INFO: 2016/11/20 20:58:05.541861 ->[172.16.0.130:6783] error during connection attempt: dial tcp4 :0->172.16.0.130:6783: getsockopt: connection refused
INFO: 2016/11/20 20:58:11.004010 ->[172.16.0.129:6783] attempting connection
INFO: 2016/11/20 20:58:11.004783 ->[172.16.0.129:6783] error during connection attempt: dial tcp4 :0->172.16.0.129:6783: getsockopt: connection refused
INFO: 2016/11/20 20:58:15.028986 ->[172.16.0.130:6783] attempting connection
INFO: 2016/11/20 20:58:15.029549 ->[172.16.0.130:6783] error during connection attempt: dial tcp4 :0->172.16.0.130:6783: getsockopt: connection refused
INFO: 2016/11/20 20:58:17.016172 ->[172.16.0.129:6783] attempting connection
INFO: 2016/11/20 20:58:17.016756 ->[172.16.0.129:6783] error during connection attempt: dial tcp4 :0->172.16.0.129:6783: getsockopt: connection refused
INFO: 2016/11/20 20:58:25.661216 ->[172.16.0.130:59100] connection accepted
INFO: 2016/11/20 20:58:25.663031 ->[172.16.0.130:59100|ea:93:cb:06:2f:a8(kube3.domain.com)]: connection ready; using protocol version 2
INFO: 2016/11/20 20:58:25.663161 overlay_switch ->[ea:93:cb:06:2f:a8(kube3.domain.com)] using fastdp
INFO: 2016/11/20 20:58:25.663315 ->[172.16.0.130:59100|ea:93:cb:06:2f:a8(kube3.domain.com)]: connection added (new peer)
INFO: 2016/11/20 20:58:25.675820 EMSGSIZE on send, expecting PMTU update (IP packet was 60028 bytes, payload was 60020 bytes)
INFO: 2016/11/20 20:58:25.676045 overlay_switch ->[ea:93:cb:06:2f:a8(kube3.domain.com)] using sleeve
INFO: 2016/11/20 20:58:25.676157 ->[172.16.0.130:59100|ea:93:cb:06:2f:a8(kube3.domain.com)]: connection fully established
INFO: 2016/11/20 20:58:25.676435 overlay_switch ->[ea:93:cb:06:2f:a8(kube3.domain.com)] using fastdp
INFO: 2016/11/20 20:58:25.683299 sleeve ->[172.16.0.130:6783|ea:93:cb:06:2f:a8(kube3.domain.com)]: Effective MTU verified at 1438
INFO: 2016/11/20 20:58:25.756375 Discovered local MAC ca:81:86:71:fa:f7
10.32.0.2
```

- Sources:
    - [Issue when using kubeadm with multiple network interfaces](https://github.com/kubernetes/kubernetes/issues/33618)
