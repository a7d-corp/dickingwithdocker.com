---
title: "Postmortem: Proxmox host drops offline twice"
date: "2025-04-04"
description: "A postmortem into why host-03 went offline"
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

A short postmortem into why a node in my Proxmox cluster went offline twice within a few days.

<!--more-->

## Issue summary

Whilst on holiday, I received a notification that a Proxmox host was offline. I was unable to access the host via SSH and the Proxmox GUI showed the host and all resident VMs as offline.

## Timeline

- 2025-03-26: `host-03` goes offline.
- 2025-03-29: `host-03` is power-cycled and comes back up.
- 2025-04-03: `host-03` goes offline again.
- 2025-04-04: after attaching a monitor it is discovered that `bond0` is down. The networking config is corrected and the host is brought back online.

## Root cause

The root cause of the issue was a misconfiguration in the network settings of `host-03`. Upon attaching a monitor after the second outage it was discovered that some spurious output from the `ip` command (see below) had somehow been appended to `/etc/network/interfaces`. I do not understand how the host was able to initially bring the networks up after booting, but the working theory as to why the node dropped offline is that an automated networking reload was tripped up by this and failed to bring `bond0` up correctly.

```bash
auto bond0
iface bond0 inet manual
	ovs_bonds enp1s0 enx60a4b758ba5b
	ovs_type OVSBond
	ovs_bridge vmbr0
	ovs_options vlan_mode=native-untagged lacp=active bond_mode=balance-tcp tag=1

auto vmbr0
iface vmbr0 inet manual
	ovs_type OVSBridge
	ovs_ports bond0 vlan1 vlan1001 vlan1002 vlan1111 vlan1000 vlan1100 vlan1200 vlan1101 vlan1102

27: enx60a4b758ba5b: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master ovs-system state UP group default qlen 1000
```

## Resolution and recovery

Removal of the spurious output from `/etc/network/interfaces` and a restart of the networking service brought `host-03` back online.

## Corrective and preventative measures

Be more careful with terminal redirection.
