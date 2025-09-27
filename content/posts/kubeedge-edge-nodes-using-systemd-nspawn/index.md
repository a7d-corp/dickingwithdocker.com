---
title: "Using systemd-nspawn containers as KubeEdge edge nodes"
date: "2025-09-27"
description: "How I'm using systemd-nspawn containers as throwaway edge nodes for testing KubeEdge."
hideSummary: true
categories:
  - "kubernetes"
tags:
  - "kubernetes"
  - "edge"
  - "kubeedge"
  - "systemd"
---

As part of my day job at [Giant Swarm](https://giantswarm.io), I've been working with [KubeEdge](https://kubeedge.io/). KubeEdge is an open-source project designed to extend Kubernetes' capabilities to hosts running at [the edge](https://en.wikipedia.org/wiki/Edge_computing). It provides core infrastructure support for networking, application deployment and metadata synchronisation between the cloud and the edge.

In order to test KubeEdge, I need edge nodes. However, I don't have any hardware I can dedicate to this purpose, and in 2025 (almost 2026), it feels somewhat overkill and unecessary to use something like a Raspberry Pi for this. I could use a VM, but I don't really want to pollute my laptop's filesystem with a bunch of packages I won't need to use all that often. As I'm running [Arch Linux](https://archlinux.org/), I already have `systemd` installed and so I can use `systemd-nspawn` for this purpose.

## What is systemd-nspawn?

`systemd-nspawn` is a tool which can be used to create and run lightweight containers. Whilst it shares the term 'container' with the likes of Docker and Podman, it is not a full containerisation platform. Instead, it is more akin to a lightweight virtual machine, providing an isolated environment for running applications. It runs a full init system with `systemd` as PID 1, so you can interact with it in a similar way to a VM, just without the overhead. Containers are also _extremely_ fast to boot, making for a short feedback loop.

## Creating a container

I wanted to be able to create new containers quickly, so I wrote a small script to do this. I also wanted to be able to test different Linux distributions, so I made the script flexible enough to allow for it - currently, it supports Arch Linux and Ubuntu. It works by using either `pacstrap` (for Arch) or `debootstrap` (for Ubuntu) to create a minimal filesystem for the container, and then configures it with a few basic settings required to run the [edgecore components](https://kubeedge.io/docs/category/edge-components) (the edge part of KubeEdge). This base directory can then be cloned to a new directory and a conainer booted from it.

The script can also facilitate joining the new edge node to an existing KubeEdge cluster (assuming your current Kubeconfig is configured correctly).

First we use the script to build the source image:

```bash
./build-container.sh --os arch --op prepare
```

Then we can clone it to a new directory, boot a container from it, and join it to the cluster:

```bash
./build-container.sh --os arch --op clone
./build-container.sh --os arch --op join
```

This is the script:

{{< gist glitchcrab 65eac34f93814bc6f6466d9a10756e6c build-container.sh >}}

Some points to note:

- [keadm](https://kubeedge.io/docs/setup/install-with-keadm/) must be installed on the host system for the `join` operation to work.

## Security considerations

`systemd-nspawn` is designed with strong isolation from the host system in mind, and so some settings have to be relaxed (or disabled) in order to allow nested containers to run. To achieve this, we need to configure the container's settings:

{{< gist glitchcrab 65eac34f93814bc6f6466d9a10756e6c container.nspawn >}}
