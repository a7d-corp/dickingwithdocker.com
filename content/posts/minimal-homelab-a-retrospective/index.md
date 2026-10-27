---
title: "Retrospective: Running a Minimal Homelab Whilst Living with the In-Laws"
date: "2026-10-27"
description: "Revisiting the mimial homelab which ran for far longer than expected.."
hideSummary: true
categories:
  - "homelab"
tags:
  - "homelab"
  - "docker"
---

A while back (over a year ago a now), I wrote a short piece about [setting up a minimal homelab](https://dickingwithdocker.com/posts/minimal-homelab-whilst-living-with-the-inlaws/) on a spare laptop to serve as a stopgap whilst we were to live with my in-laws as part of moving house. At the time, I said it was to be for an indeterminate period of time (likely a couple of months). Unfortunately I appear to have jinxed it as we have actually been living with them for over a year now. There's no need to bore you with the how and why this ended up happening, but it did mean that my temporary homelab became a lot more permanent than I had anticipated.

My full lab runs a bunch of services which the household depends on, but at the same time it also serves as a playground for me to experiment with new technologies and tools. I do this because I like to mess around with new  shiny things, and it also serves to keep me sharp (especially in this strange new AI-lead world we seem to have found ourselves in).

Losing this capability has felt like losing an arm at times, and so naturally my temporary setup has evolved somewhat in an attempt to at least partially fill this hole. I'll use this post to go over what worked, what didn't, and what I've added.

## What worked

The original plan involved keeping the setup pretty simple; Caddy for access, Plex for media, Blocky for DNS and Tailscale for access.

All of these worked as expected - I've run them for a long time and I'm very familiar with them. The only issue I ran into was using Tailscale to access media when I'm outside the house; the in-law's internet connection isn't as fast as I was used to and in order to stream media it was necessary to reduce the bitrate significantly.

In order to use Blocky for DNS-level adblocking, the laptop needed to have a static IP. I picked one near the top of the subnet in the hope that it would be outside the router's configured DHCP range - this seems to have worked fine as I've had no address conflicts at all.

Overall, keeping it simple and just going with Ubuntu Server + Docker worked well (until it became too restrictive, which we'll cover in a bit).

## What changed

Nothing (well, nothing got taken away). But given that it was a fairly bare-bones setup to start with, that's not all that surprising.

## What's new

This is where things start to get a bit busy. Having my main sandbox offline for this long has meant that this temporary laptop lab has been somewhat abused and extended far beyond what I originally envisioned. The current count is 27 running containers - way beyond the original 3. I won't cover all of them, but there are a few worth talking about.

#### The Arr Stack

I've been running Plex for years, but over time I've become a little disillusioned with some of the decisions they've taken. Whilst looking for something to experiment with one weekend, I decided to run Jellyfin, and this ended up snowballing somewhat. I'm not quite ready to move away from Plex yet (at least, not until we're moved in and all the services in the house are running again), so I also set up [a sync container](https://github.com/luigi311/JellyPlex-Watched) to ensure that the watch status is replicated both ways, regardless of where the content is viewed.

Whilst I was overhauling my media setup, I decided that I should really check out the [Arr Stack](https://wiki.servarr.com/) too (yes I realise I'm a bit late to the party). Anything which improves the [Wife Approval Factor](https://en.wikipedia.org/wiki/Wife_acceptance_factor) can only be a good thing - if it reduces the friction involved in obtaining new Linux ISOs then it's worth doing. I settled on the 'core' stack plus a couple of extras ([prowlarr](https://wiki.servarr.com/prowlarr), [sonarr](https://wiki.servarr.com/sonarr), [radarr](https://wiki.servarr.com/radarr), [bazarr](https://www.bazarr.media/) and [profilarr](https://github.com/Dictionarry-Hub/Profilarr/)). After running this setup for a little while I also added [flaresolverr](https://github.com/FlareSolverr/FlareSolverr), [seerr](https://seerr.dev/), [posterizarr](https://github.com/fscorrupt/posterizarr) and [cleanuparr](https://github.com/Cleanuparr/Cleanuparr). So far, this seems to be working very well for me and it's left me wondering why I didn't listen to an old colleague and set it up years ago (sorry Eric!)

#### iCloud backups

When my homelab is running, I have an older Mac Mini which is used purely to run a script which extracts my iCloud photos from iPhoto and dumps them onto my NAS. I was OK with not running this for a couple of months, but as time dragged on I decided I needed to do something about this. In its place, I've been running [iCloudPD](https://github.com/boredazfcuk/docker-icloudpd) which stores them on an external drive attached to the laptop. Aside from having to occasionally re-authenticate when its cookie expires, it's been pretty seamless. The initial download of several hundred gigabytes of photos took a long time, but once this was complete the updates are incremental and pose no issues. It's worth it for the peace of mind it gives me.

#### Frigate NVR

I've always had a bunch of CCTV cameras around the house and I use [Frigate](https://github.com/blakeblackshear/frigate) to aggregate the feeds, detect and track objects etc. As most of the cameras are in a box at the moment I had no need for it, however I do have a camera in my young son's room. When he went through a period of very disturbed sleep, I decided to set Frigate up as I knew it would be very useful to able to go back through the events overnight and see how long he had been awake for. Is Frigate overkill for a single camera? Sure, but I was already very familiar with it so it just made sense. Any friction removed can only be a good thing.

#### S3 storage

Whilst my homelab has been offline, [Minio pulled the rug out from under everyone using the free version](https://github.com/minio/minio/commit/27742d469462e1561c776f88ca7a1f26816d69e2). As I use (well, used) Minio for storage in my lab, I knew that when everything is up and running again I will need to replace Minio with something else. After hunting around for alternatives, I settled on [seaweedfs](https://github.com/seaweedfs/seaweedfs). I don't actually need it for any services running on the laptop, but it seems like getting familiar with it would be a good idea.

#### Running VMs

Wait, what? Isn't this post about containers? Yes, it is, however not everything fits neatly into the container paradigm, and I was missing the ability to run fully-fledged VMs. After some research, I found [qemus/qemu](https://github.com/qemus/qemu), which is a very nice wrapper around running QEMU-based VMs inside a container. You just need to set an environment variable on the container to select [which OS you want to run](https://github.com/qemus/qemu#how-do-i-select-the-operating-system) (you can also run any custom image you want, it just needs to be in a supported format). Exposing ports is simple enough using Docker's port proxying ability - in order to SSH to the first VM I set up I can just use port `2222` + the laptop's IP and this is proxied through to the Debian VM in the container. Additionally, the VM is stateful (as long as you configure the VM storage directory as a host volume mount). On initial container startup, a root disk for the VM is provisioned in this storage and the VM boots from it every time the container is restarted thereafter. It's not a replacement for Proxmox, but it bridges the gap for now.

## In conclusion

Overall, I've been pretty pleased with the whole setup. Using an old laptop worked pretty well as it's a relatively small form factor whilst still having enough grunt to run all these applications. When setting up Plex (and later Jellyfin), being able to use the onboard GPU for transcoding has reduced the load on the hardware whilst also improving the end user experience.

Things definitely got more complex than I was originally planning, but each addition solved a specific problem. Plus it gave me something to focus on and kept me quiet - my wife would definitely say that was an added bonus.
