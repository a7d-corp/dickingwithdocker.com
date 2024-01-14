---
title: "Update: Using BGP to integrate Cilium with OPNsense"
date: "2024-01-14"
description: "An update to integrating Cilium and OPNsense using BGP"
categories:
  - "kubernetes"
  - "networking"
  - "bgp"
tags:
  - "containers"
  - "kubernetes"
  - "networking"
  - "homelab"
  - "bgp"
---

A little while back, I wrote a [short piece on integrating Cilium with OPNsense using BGP](/posts/using-bgp-to-integrate-cilium-with-opnsense/). With more recent releases of Cilium, the team have introduced the [Cilium BGP Control Plane](https://docs.cilium.io/en/stable/network/bgp-control-plane/) (currently as a beta feature). This reworking of the BGP integration replaces the old MetalLB-based control plane and as such the older feature must first be disabled. To enable the new feature, you can either pass an argument to Cilium:

```shell
--enable-bgp-control-plane=true
```

Or if you use Helm to install Cilium then the following values are required:

```yaml
bgpControlPlane:
  enabled: true
```

Note that the OPNsense configuration described in the [previous post](/posts/using-bgp-to-integrate-cilium-with-opnsense/) is unchanged; it is only the configuration of Cilium which differs.

## BGP configuration

Configuration of the BGP feature is somewhat different to the single `bgp-config` ConfigMap used by the old MetalLB integration. Below is an example of the old configuration which we'll translate into the new Custom Resources provided by Cilium:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bgp-config
  namespace: cilium
data:
  config.yaml: |
    peers:
      - peer-address: 172.25.0.1
        peer-asn: 65401
        my-asn: 65502
    address-pools:
      - name: default
        protocol: bgp
        addresses:
          - 172.27.0.32-172.27.0.62
```

### CiliumLoadBalancerIPPool

The address pool is now configured by way of the new `CiliumLoadBalancerIPPool` Custom Resource. This CR separates out the address configuration from the BGP router configuration:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: service-pool
spec:
  cidrs:
  - cidr: 172.27.0.32/27
```

### CiliumBGPPeeringPolicy

The `CiliumBGPPeeringPolicy` CR is a little more complex, however most of the `neighbors` section is just the defaults and so they can be ignored:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
 name: bgp-peering-policy-worker
spec:
 nodeSelector:
   matchLabels:
     metal.sidero.dev/serverclass: worker
 virtualRouters:
 - localASN: 65502
   serviceSelector:
     matchExpressions:
       - {key: "io.cilium/bgp-announce", operator: In, values: ['worker']}
   neighbors:
    - peerAddress: '172.25.0.1/32'
      peerASN: 65401
      eBGPMultihopTTL: 10
      connectRetryTimeSeconds: 120
      holdTimeSeconds: 90
      keepAliveTimeSeconds: 30
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 120
```

Some sections of the peering policy require a little more explanation.

Firstly, I only have a single neighbor defined as my OPNsense firewall is the only peer in my network. This is easily translated over from the old configuration, along with the local ASN for the cluster:

```yaml
  virtualRouters:
  - localASN: 65502
    neighbors:
     - peerAddress: '172.25.0.1/32'
       peerASN: 65401
```

The new peering policy resource allows far more flexibility for managing BGP topologies. In my case, I only want to allow worker nodes to announce routes, so I've configured the peering policy to only apply to worker nodes. This is achieved by applying a label to those nodes which is then used in the policy:

```yaml
  nodeSelector:
    matchLabels:
      metal.sidero.dev/serverclass: worker
```

The final config option to note is the `serviceSelector`; I do not want every Service IP to be announced so I ensure that Cilium only announces those which I want it to. In order to achieve this I label any Services which should be announced with `io.cilium/bgp-announce=worker`:

```yaml
   serviceSelector:
     matchExpressions:
       - {key: "io.cilium/bgp-announce", operator: In, values: ['worker']}
```

I chose this scheme because I also have a peering policy which only applies to control plane nodes.

### BGP topologies

Having only one upstream peer, my setup is quite simple. It is possible to create more complex topologies; for these I suggest [referring to the documentation](https://docs.cilium.io/en/stable/network/bgp-control-plane/#creating-a-bgp-topology).
