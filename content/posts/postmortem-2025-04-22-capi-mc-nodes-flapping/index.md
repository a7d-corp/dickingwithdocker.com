---
title: "Postmortem: Management Cluster nodes flapping"
date: "2025-04-22"
description: "A postmortem into flapping CAPI cluster nodes"
categories:
  - "homelab"
  - "proxmox"
  - "kubernetes"
tags:
  - "proxmox"
  - "homelab"
  - "cluster-api"
  - "kubernetes"
  - "postmortem"
---

A postmortem around sporadic flapping nodes in my ClusterAPI management cluster.

<!--more-->

## Issue summary

After pivoting management of cluster `room101-a7d-mc` from the bootstrap cluster to the management cluster, nodes began flapping and intermittently dropping offline.

```
➜  talosctl get nodestatuses
NODE            NAMESPACE   TYPE         ID                                   VERSION   READY   UNSCHEDULABLE
172.25.100.10   k8s         NodeStatus   room101-a7d-mc-cp-e2cc5a-2cwsv       80        true    false
172.25.100.11   k8s         NodeStatus   room101-a7d-mc-cp-e2cc5a-mw5js       47        false   false
172.25.100.12   k8s         NodeStatus   room101-a7d-mc-cp-e2cc5a-rq5cw       52        true    false
172.25.100.13   k8s         NodeStatus   room101-a7d-mc-workers-gvh48-v6hrz   178       true    false
172.25.100.14   k8s         NodeStatus   room101-a7d-mc-workers-gvh48-k6v2r   97        true    false
```

## Timeline

- `21-04-2025`: CAPI MC was promoted to self-managing (pivoted from bootstrap cluster)
- `21-04-2025`: HelmRelease for `kube-prometheus-stack` was applied (maybe related)
- `21-04-2025`: nodes start being flagged as `NotReady`
- `22-04-2025`: `apiserver` was observed being oomkilled on a master

## Logs

```
172.25.100.11: kern:     err: [2025-04-22T10:37:36.727637155Z]: Out of memory: Killed process 2735 (kube-apiserver) total-vm:3112352kB, anon-rss:1505564kB, file-rss:104kB, shmem-rss:0kB, UID:65534 pgtables:3800kB oom_score_adj:827
```

## Root cause

It was observed that an affected node was seeing OOMKill events on the `api-server` process.

## Resolution and recovery

Increase the control plane RAM allocation from 3Gb to 5Gb and the worker node RAM allocation from 2Gb to 4Gb.

## Future preventative measures

It would be easier to diagnose issues such as this with centralised log aggregation from the nodes. This could be achieved by setting up a log server and [configuring the nodes to ship logs to it](https://www.talos.dev/v1.9/reference/configuration/v1alpha1/config/#Config.machine.logging).
