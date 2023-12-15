---
title: "Deploying Kubernetes on VMs with Kubespray"
date: "2017-08-09"
categories: 
  - "ansible"
  - "automation"
  - "containers"
  - "deployment"
  - "kubernetes"
  - "orchestration"
tags: 
  - "ansible"
  - "automation"
  - "config-management"
  - "deployment"
  - "kubernetes"
  - "orchestration"
---

### All the choices

So you're looking to start using Kubernetes, but you're overwhelmed by the multitude of deployment options available? Judging by the length of the [Picking the Right Solution](https://kubernetes.io/docs/setup/pick-right-solution/) section to the Kubernetes docs, it's safe to assume that you're not alone. Even after you've made it past the provisioning stage, you then need to learn how to administrate what is a very complex system. In short; Kubernetes is not easy.

Naturally then, the easiest way to use Kubernetes is to let someone else look after the infrastructure for you. If you're looking for the most hands-off turnkey solution then you should investigate Google's [Container Engine](https://cloud.google.com/container-engine/), or RedHat's [OpenShift Dedicated](https://www.openshift.com/dedicated/) platform.

Whilst these solutions do make it easier to consume Kubernetes for your workloads by abstracting away the management overheads, you may well need to maintain greater control over your data. Whilst using the cloud is perfectly safe, this may not be something your security controls permit, making cloud-hosted options such as those mentioned above unviable. In these cases, unless you have a large budget to spend on something like RedHat's [on-premise Container Platform](https://www.openshift.com/container-platform/), you'll need to look at rolling your own.

At this point, [picking the right solution](https://kubernetes.io/docs/setup/pick-right-solution/) has become a _little_ easier. As with all open-source projects, it's important to judge their suitability - you don't want to adopt an unmaintained project unless you have the time and resources to maintain it yourself. This is especially important when interacting with Kubernetes, as the development moves fast and new features are added regularly.

With all this in mind, I ended up settling on [Kubespray](https://github.com/kubernetes-incubator/kubespray) (originally called Kargo). Kubespray is a Kubernetes [incubator](https://github.com/kubernetes/community/blob/master/incubator.md) project, which means it is on its way to becoming a fully-fledged community project. I spend a fair bit of my own time working with Ansible, and as Kubespray is just a large set of playbooks, it was the obvious choice.

### Getting started

The rest of this post assumes that your target hosts are already correctly configured for administration via Ansible - if you're starting off with virgin nodes then you can use something like my [Ansible node bootstrapping](http://dickingwithdocker.com/2017/08/ansible-node-bootstrapping/) playbook. Obviously this still requires access of some fashion; in my case my hosting provider [Memset](https://www.memset.com) gives you the ability to inject an SSH key at provisioning time. Once this is in place I can run my playbook to reconfigure the node as I see fit.

##### Configuring your hosts file

Now you're ready to start, you'll obviously need to clone the Kubespray repo. Once you've done this, duplicate the inventory directory and jump there. You'll need to configure your inventory file to suit your deployment; below is mine for illustration's sake:

```
[all]
node1   ansible_ssh_host=1.2.3.1    ip=10.1.205.11
node2   ansible_ssh_host=1.2.3.2    ip=10.1.205.12
node3   ansible_ssh_host=1.2.3.3    ip=10.1.205.13
node4   ansible_ssh_host=1.2.3.4    ip=10.1.205.14
node5   ansible_ssh_host=1.2.3.5    ip=10.1.205.15

[kube-master]
node1
node2

[kube-node]
node1
node2
node3
node4
node5

[etcd]
node3
node4
node5

[k8s-cluster:children]
kube-node
kube-master

[calico-rr]
```

The `[all]` group needs to contain all nodes in the cluster. If you wish to have them communicate over a different IP to the access IP then you can specify this using the `ip` variable. In my case, I have a VLAN between all 5 nodes and I want my internal cluster traffic to traverse this, rather than the public internet.

The nodes listed under the `[kube-master]` heading will become cluster [masters](https://kubernetes.io/docs/admin/high-availability/#master-elected-components) who's job is to modify the cluster's state by scheduling pods etc. For HA purposes, it's a good idea to have more than one.

Any node in the `kube-node` group will be available for scheduling pods onto. In a larger cluster these nodes will purely function as minions, however in my case my masters are also minions too.

Finally, the `[etcd]` group tells Ansible which nodes to deploy your [etcd](https://github.com/coreos/etcd) cluster onto. Kubernetes stores a lot of information in etcd so three nodes is good practise. You could run with a single etcd node, but if that goes away then the entire cluster will grind to a halt.

##### Global config options

Contained within your duplicated inventory directory is a `group_vars` directory. Assuming you're familiar with Ansible the purpose of this directory is clear. Unless you require some edge cases, the majority of `all.yml` can remain as provided:

```
# Valid bootstrap options (required): ubuntu, coreos, centos, none
bootstrap_os: ubuntu

### OTHER OPTIONAL VARIABLES
## For some things, kubelet needs to load kernel modules.  For example, dynamic kernel services are needed
## for mounting persistent volumes into containers.  These may not be loaded by preinstall kubernetes
## processes.  For example, ceph and rbd backed volumes.  Set to true to allow kubelet to load kernel
## modules.
kubelet_load_modules: true
```

Nothing too complicated going on here; one thing to bear in mind here is that the module loading isn't persistent. Ansible will modprobe any modules Kubernetes needs, but this won't survive a reboot (as I discovered the hard way).

##### Cluster config options

The `k8s-cluster.yml` file requires a little more customisation. Firstly, I'd suggest disabling anonymous authentication to the API for security reasons. You then need to pick the version of Kubernetes to deploy, and set some passwords:

```
kube_api_anonymous_auth: false

## Change this to use another Kubernetes version, e.g. a current beta release
kube_version: v1.6.7

# Users to create for basic auth in Kubernetes API via HTTP
kube_api_pwd: "YOUR PASSWORD HERE"
kube_users:
  kube:
    pass: "{{kube_api_pwd}}"
    role: admin
  root:
    pass: "{{kube_api_pwd}}"
    role: admin
```

Next up, some networking config. As is to be expected, there are a multitude of options when it comes to setting up [networking in Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/networking/). Kubespray gives you a subset of these options; I chose to go with [Weave](https://www.weave.works/docs/net/latest/overview/) as it means I can deploy [Weave Scope](https://www.weave.works/oss/scope/) at a later date.

```
# Choose network plugin (calico, weave or flannel)
# Can also be set to 'cloud', which lets the cloud provider setup appropriate routing
kube_network_plugin: weave

# Kubernetes internal network for services, unused block of space.
kube_service_addresses: 10.233.0.0/18

# internal network. When used, it will assign IP
# addresses from this range to individual pods.
# This network must be unused in your network infrastructure!
kube_pods_subnet: 10.240.00.0/18
```

Lastly, we have some options it's worth enabling which help visualise the state of your cluster. The [Netchecker](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/netcheck.md) deployment places pods on all hosts which constantly verify the state of the networking by attempting to communicate with each other.

```
# Deploy netchecker app to verify DNS resolve as an HTTP service
deploy_netchecker: true

# Monitoring apps for k8s
efk_enabled: true
```

### Deployment

With everything configured to your liking, it's time to deploy to your nodes. In the root of the Kubespray checkout is `cluster.yml`; this is what we'll need to use. Deployment is carried out the usual Ansible way:

```
$ ansible-playbook -i kube_cluster/hosts cluster.yml -b -v
```

The run-time of this playbook is very much dependent on a number of variables, such as the number of nodes and also how much Internet-connected bandwidth they have; for my nodes, it took around 20 minutes.

### Accessing your cluster

Before you can use your cluster, you need to configure access to it. During the cluster bootstrapping process, various SSL certificates will have been created which we can use to authenticate ourselves. From your master node in `/etc/kubernetes/ssl` you'll need to grab a copy of the node's admin certificate, the corresponding private key, and the CA certificate. You'll need to place these in `/home/user/.kube/` on the machine you wish to use to administrate the cluster, along with a config file pointing to them:

```
$ cat .kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/user/.kube/ca.pem
    server: https://kubernetes.default:6443
  name: kargo
contexts:
- context:
    cluster: kargo
    user: admin
  name: kargo
current-context: kargo
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate: /home/user/.kube/admin-node1.pem
    client-key: /home/user/.kube/admin-node1-key.pem
```

Finally, ensure the `server:` variable is a URL by which you can talk to your master node.

##### Verify the cluster state

At this point, you should be able to run `kubectl get pods -o wide --all-namespaces` (note that you may see more or less pods if you chose different options in your config).

```
NAMESPACE     NAME                                    READY     STATUS    RESTARTS   AGE       IP            NODE
default       netchecker-agent-7bnqq                  1/1       Running   0          13d       10.240.26.1   node3
default       netchecker-agent-hostnet-4xhlc          1/1       Running   1          14d       10.1.205.14   node5
default       netchecker-agent-hostnet-gb2wf          1/1       Running   1          14d       10.1.205.11   node2
default       netchecker-agent-hostnet-ld44s          1/1       Running   1          14d       10.1.205.10   node1
default       netchecker-agent-hostnet-xw5r2          1/1       Running   1          14d       10.1.205.12   node3
default       netchecker-agent-hostnet-zdxrz          1/1       Running   1          14d       10.1.205.13   node4
default       netchecker-agent-nhlkp                  1/1       Running   0          13d       10.240.10.1   node4
default       netchecker-agent-qr5zg                  1/1       Running   0          13d       10.240.14.0   node2
default       netchecker-agent-xs8xm                  1/1       Running   0          13d       10.240.8.1    node5
default       netchecker-agent-ztp40                  1/1       Running   0          13d       10.240.30.0   node1
default       netchecker-server-3646041304-gzl3m      1/1       Running   18         13d       10.240.8.4    node5
kube-system   kube-apiserver-node1                    1/1       Running   1          13d       10.1.205.10   node1
kube-system   kube-apiserver-node2                    1/1       Running   1          13d       10.1.205.11   node2
kube-system   kube-controller-manager-node1           1/1       Running   2          13d       10.1.205.10   node1
kube-system   kube-controller-manager-node2           1/1       Running   1          13d       10.1.205.11   node2
kube-system   kube-dns-3841192733-7nbbx               3/3       Running   1          13d       10.240.8.5    node5
kube-system   kube-dns-3841192733-7zg62               3/3       Running   0          13d       10.240.26.4   node3
kube-system   kube-proxy-node1                        1/1       Running   1          13d       10.1.205.10   node1
kube-system   kube-proxy-node2                        1/1       Running   1          13d       10.1.205.11   node2
kube-system   kube-proxy-node3                        1/1       Running   1          13d       10.1.205.12   node3
kube-system   kube-proxy-node4                        1/1       Running   1          13d       10.1.205.13   node4
kube-system   kube-proxy-node5                        1/1       Running   1          13d       10.1.205.14   node5
kube-system   kube-scheduler-node1                    1/1       Running   3          13d       10.1.205.10   node1
kube-system   kube-scheduler-node2                    1/1       Running   2          13d       10.1.205.11   node2
kube-system   kubedns-autoscaler-1833630871-rznl5     1/1       Running   0          13d       10.240.26.3   node3
kube-system   kubernetes-dashboard-2039414953-s4ts8   1/1       Running   5          13d       10.240.8.6    node5
kube-system   monitoring-grafana-2527507788-5cbbd     1/1       Running   0          11d       10.240.14.2   node2
kube-system   monitoring-influxdb-3480804314-6zw45    1/1       Running   0          11d       10.240.10.3   node4
kube-system   nginx-proxy-node3                       1/1       Running   1          13d       10.1.205.12   node3
kube-system   nginx-proxy-node4                       1/1       Running   1          13d       10.1.205.13   node4
kube-system   nginx-proxy-node5                       1/1       Running   1          13d       10.1.205.14   node5
kube-system   weave-net-2t7jp                         2/2       Running   2          14d       10.1.205.14   node5
kube-system   weave-net-45h6g                         2/2       Running   2          14d       10.1.205.10   node1
kube-system   weave-net-c6bbl                         2/2       Running   3          14d       10.1.205.13   node4
kube-system   weave-net-lh6x2                         2/2       Running   2          14d       10.1.205.11   node2
kube-system   weave-net-s43ct                         2/2       Running   2          14d       10.1.205.12   node3
```

Provided all pods are listed as running then you should have a healthy cluster. The most complete way to administrate your cluster is using kubectl, however it's not great for at-a-glance monitoring. Fortunately, Kubernetes has you covered with their dashboard. You can deploy it straight from the git repo:

```
$ kubectl create -f https://git.io/kube-dashboard
```

With that done, you will need to grab a terminal as you'll use kubectl to proxy the dashboard from the cluster:

```
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

Finally, open your favourite browser and head to [http://localhost:8001/ui](http://localhost:8001/ui) and you should see the dashboard. Success!

### Bonus round

If you went with [Weave](https://www.weave.works/docs/net/latest/overview/) for your networking, you can also deploy [Weave Scope](https://www.weave.works/oss/scope/) which gives you a great deal of insight into your cluster. Unsurprisingly, we deploy on top of Kubernetes which means no need to configure anything - Weave will automatically provide the metrics to the Scope pods.

```
$ kubectl apply --namespace kube-system -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\\n')"
```

With that done, you'll then use kubectl to forward the relevant port:

```
$ kubectl port-forward -n kube-system "$(kubectl get -n kube-system pod --selector=weave-scope-component=app -o jsonpath='{.items..metadata.name}')" 4040
```

And then as for the dashboard, open up your browser and navigate to [http://127.0.0.1:4040](http://127.0.0.1:4040).
