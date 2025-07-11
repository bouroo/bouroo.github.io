---
title: "Linux Sysctl Tuning"
subtitle: ""
date: 2023-07-19T23:56:31+07:00
lastmod: 2023-07-19T23:56:31+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Tune sysctl settings to make Linux servers run smoothly under increased load."
aliases:
- /posts/linux_sysctl_tuning/
license: ""
images: []
resources:
- name: "featured-image"
  src: "featured-image.jpg"

tags: ["Linux", "DevOps", "PVE", "K3S", "K8S"]
categories: ["Linux", "DevOps"]

featuredImage: "featured-image.jpg"
featuredImagePreview: "featured-image"
---

We can adjust sysctl settings to make Linux servers run smoothly under increased load. Normally, each Linux distribution has a default sysctl configuration. For example, RHEL might be optimized for server use, while DEB might be balanced for general performance. In this article, I will introduce the settings I use in production for each type of workload:

<!--more-->

## Sysctl for General Servers `60-sysctl.conf`
> You can place these settings in `/etc/sysctl.d/60-sysctl.conf`

{{< gist bouroo bc52ad58a6e75d44e5235b229e9ca988 60-sysctl.conf >}}

## Additional Sysctl for Proxmox VE `80-pve.conf`
> You can place these settings in `/etc/sysctl.d/80-pve.conf`

{{< gist bouroo bc52ad58a6e75d44e5235b229e9ca988 80-pve.conf >}}

## Additional Sysctl for K3S, K8S `80-k8s-ipvs.conf`
> You can place these settings in `/etc/sysctl.d/80-k8s-ipvs.conf`

{{< gist bouroo bc52ad58a6e75d44e5235b229e9ca988 80-k8s-ipvs.conf >}}

## Apply Sysctl Settings
```bash
sysctl --system
```
> For machines running containerd, k3s, or k8s, you must also restart the containerd, k3s, or k8s services.
