---
title: "Over-engineering my website with Kubernetes"
date: "2018-03-18"
categories: 
  - "ansible"
  - "deployment"
  - "docker"
  - "kubernetes"
  - "orchestration"
  - "packer"
  - "provisioning"
tags: 
  - "ansible"
  - "automation"
  - "containers"
  - "deployment"
  - "kubernetes"
  - "orchestration"
---

## A solution in need of a problem

Like all good sysadmins, my personal website has been a 'coming soon' splash page for quite some time. According to the [Wayback Machine](https://web.archive.org/web/20140401000000*/simonweald.com), it's been this way since some time in 2014. As I'm sure many can sympathise with, there are always far more interesting and shiny things to be experimenting with than building a website.

One of the interesting things I like to experiment with is Kubernetes (as should be apparent from the tag cloud). Up until now, this has mostly consisted of [building clusters](/2017/08/deploying-kubernetes-vms-kubespray/), tweaking them and then tearing them down again. Whilst this gives me experience from the Operations side, I'm not getting the end-to-end experience a consumer of my cluster would have.

Having a website to deploy to Kubernetes then seems to solve both problems. Once the site is up and running the challenge becomes supporting the cluster as if it were in production. Eventually the aim is to have pushes to the site's [Github repo](https://github.com/analbeard/simonweald.com) trigger a container build/test pipeline with Jenkins which then updates the Kubernetes deployment.

## My toolbox

In order to build a Kubernetes deployment, I first need to turn my site into a Docker container. Using [Hugo](https://gohugo.io/), the static site files can be collected using a simple `hugo` command in the `/site/` directory. The container image is built using [Packer](https://www.packer.io/) and is defined in the [build](https://github.com/analbeard/simonweald.com/blob/master/build.json) file - in this instance I'm using the official [Nginx Alpine](https://hub.docker.com/_/nginx/) image as it's nice and lightweight.

Customisation of the image is carried out by [Ansible](https://www.ansible.com/) as described in the [provision playbook](https://github.com/analbeard/simonweald.com/blob/master/provision.yaml). This transfers the static site files and the Nginx config into the image. This is then pushed to my private Docker registry.

At this point, the site is ready to be deployed to the cluster. As it is 2018, the site should definitely be TLS encrypted. To achieve this, a combination of [Heptio's Contour](https://github.com/heptio/contour) (which uses the fantastic [Envoy proxy](https://www.envoyproxy.io/) underneath) and [Jetstack's cert-manager](https://github.com/jetstack/cert-manager) is used. Using cert-manager allows the cluster to leverage automatically renwed [Let's Encrypt](https://letsencrypt.org/) certificates.

The Kubernetes [yaml files](https://github.com/analbeard/simonweald.com/blob/master/kubernetes/) are all pretty vanilla, but using Contour means I can apply the `ingress.kubernetes.io/force-ssl-redirect: "true"` annotation to the ingress resource. This automatically upgrades all HTTP connections to HTTPS by default.

Tying all these tools together was a useful learning experience with a tangible result, which can be seen at [simonweald.com](https://www.simonweald.com/) (provided I haven't broken anything since).
