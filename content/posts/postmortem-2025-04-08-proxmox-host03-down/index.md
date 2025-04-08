---
title: "Postmortem: Proxmox host goes offline"
date: "2025-04-04"
description: "Postmortem into why host-03 went offline twice"
categories:
  - "homelab"
  - "proxmox"
  - "networking"
tags:
  - "proxmox"
  - "homelab"
  - "sysadmin"
  - "networking"
  - "postmortem"
---

A postmortem into why a node in my Proxmox cluster went offline twice within a few days.

# Postmortem: Proxmox host-03 goes offline twice

## Issue summary

Whilst on holiday, I received a notification that a Proxmox host was offline. I was unable to access the host via SSH and the Proxmox GUI showed the host and all resident VMs as offline.

## Timeline

- 2025-03-26: `host-03` goes offline.
- 2025-03-29: `host-03` is power-cycled and comes back up.
- 2025-04-03: `host-03` goes offline again.
- 2025-04-04: after attaching a monitor it is discovered that `bond0` is down. The networking config is corrected and the host is brought back online.

## Root cause

The root cause of the issue was a misconfiguration in the network settings of `host-03`. Upon attaching a monitor after the second outage it was discovered that some spurious output from the `ip` command (see below) had somehow been appended to `/etc/network/interfaces`. The working theory is that an automated networking reload was tripped up by this and failed to bring `bond0` up correctly.

```bash
27: enx60a4b758ba5b: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master ovs-system state UP group default qlen 1000
```

## Resolution and recovery

Removal of the spurious output from `/etc/network/interfaces` and a restart of the networking service brought `host-03` back online.

## Corrective and preventative measures

Be more careful with terminal redirection.
