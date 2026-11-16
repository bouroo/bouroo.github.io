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
  src: "featured-image.svg"

tags: ["Linux", "DevOps", "PVE", "K3S", "K8S"]
categories: ["Linux", "DevOps"]

featuredImage: "featured-image.svg"
featuredImagePreview: "featured-image.svg"
---

sysctl settings control kernel limits on connections, memory, and filesystem buffers, and the values that ship with a distribution target a general machine. These are the settings I use in production, split into three files by workload: general servers, Proxmox VE, and K3S/K8S with IPVS.

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
