---
title: "Exposing Web Services to the World with K3S + Cloudflare Tunnel"
subtitle: ""
date: 2023-07-20T19:49:54+07:00
lastmod: 2023-07-20T19:49:54+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "This article will guide you on how to expose your web services to the outside world with K3S + Cloudflare Tunnel without needing an External Public IP."
aliases:
- /posts/behind_cloudflare/
license: ""
images: []
featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

tags: ["Cloudflare", "DevOps", "K3S", "K8S"]
categories: ["DevOps"]

lightgallery: true
---

If you have a home lab or a service in K3S/K8S that you want to expose to the world, but you don't have a public IP, what can you do? Especially in today's world where ISPs often provide Carrier-grade NAT, or what's commonly known as large-scale NAT (LSN), making it difficult to update IPs with DDNS. This is where the hero of this article comes in.

<!--more-->

## What is Cloudflare Tunnel?
![Access Diagram](img/Access_Diagram.webp "Access Diagram")
Cloudflare Tunnel, as the name suggests, creates a tunnel from your network to the Cloudflare network, making Cloudflare the front-end between you and the outside world.

## Creating a Cloudflare Tunnel
> You need to have at least one domain linked to Cloudflare.

Start by enabling it at https://one.dash.cloudflare.com/
Then go to Access > Tunnels > Create a tunnel
![create_tunnel](img/create_tunnel.webp "create_tunnel")

Name your Tunnel
![naming_tunnel](img/naming_tunnel.webp "naming_tunnel")

Get the tunnel token to put into a kube secret when deploying.
![tunnel_token](img/tunnel_token.webp "tunnel_token")

## Deploying Cloudflare Tunnel on K3S/K8S

Create a `cloudflared-daemonset.yml` file like this, by converting the tunnel token to base64 and putting it in a secret named `cf_tunnel_token`.
{{< gist bouroo 624c6cd6d515c5e0af54904dba60f073 cloudflared-daemonset.yml >}}

Then deploy with:
```bash
kubectl apply -f cloudflared-daemonset.yml
```

## Exposing services to the outside world
Check the status in the Cloudflare One dashboard to confirm that your tunnel is connected.
![tunnel_config](img/tunnel_config.webp "tunnel_config")

It's like having a kube-proxy + load balancer all in one!
![kube_proxy](img/kube_proxy.webp "kube_proxy")

Then we can reverse proxy to our service in the cluster, for example, `service_name.namespace`, such as `https` to `rancher` on the `cattle-system` namespace, as shown in the image.
![ingress](img/ingress.webp "ingress")
If the service uses a self-signed SSL, we need to tell Cloudflare to skip verification.
![tls_skip_verify](img/tls_skip_verify.webp "tls_skip_verify")


## Testing the service access
![rancher](img/rancher.webp "rancher")
![daemonset](img/daemonset.webp "daemonset")

