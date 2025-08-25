---
title: "Running a Minimal Homelab Whilst Living with the In-Laws"
date: "2025-08-24"
description: "A short post about hacking together a minimal homelab whilst living with the in-laws."
hideSummary: true
categories:
  - "homelab"
tags:
  - "homelab"
  - "docker"
---

We’re in the middle of moving house, and due to various _shenanigans_ with our [property chain](https://www.halifax.co.uk/mortgages/help-and-advice/what-is-a-property-chain.html), we have ended up needing to live with my in-laws for an indeterminate length of time (likely 2-3 months). This means that the homelab has been boxed up and stuck in storage. Whilst a lot of what I run day-to-day isn't crucial and I can cope without it, there are a few services which will make the next few months a little more bearable.

The setup is deliberately lightweight, I'm using a Dell XPS laptop with Ubuntu Server running Docker. I did toy with the idea of running Proxmox on it but the networking started getting a little complex due to the need to use the laptop's wifi to connect to the network.

### Containers

#### Caddy

I use Caddy everywhere already so it made sense to use it here too. With Caddy I can have TLS-terminated connections with trusted certificates from LetsEncrypt provisioned via the DNS01 challenge method.

#### Plex

I already use Plex so it was trivial to set up a second instance. Prior to powering off my main NAS I dumped a ton of media onto a large external hard drive which is then served via this temporary Plex instance. I've configured a DNS A record for Plex to point to the laptop's static IP (see below) which means it is also reachable via Tailscale (also see below).

#### Blocky

We have network-level ad-blocking in place at home and I felt that this was one thing I couldn't live without - it makes the internet feel so much cleaner. The server runs Blocky and our phones are configured to use the laptop's static IP (see below) for DNS when we're on the in-law's wifi.

#### Tailscale

I already run Tailscale everywhere so it made sense to run it here too - the laptop is configured as an exit for the local subnet so Plex etc is accessible from anywhere.

### Other points of note

- I have no control over the network's configuration so I just picked an IP high up in the subnet and configured it statically. Hopefully this should avoid any conflicts.
- I've disconnected the battery as it will be permanently plugged in and I don't fancy causing a spicy firey pillow.

### Summary

This is certainly not up to my usual level of over-engineering, but it should serve (hurr) the purpose for now. And I'm also somewhat proud of what I've pulled together in a short space of time too.
