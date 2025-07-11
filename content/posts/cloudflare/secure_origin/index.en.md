---
title: "Hiding Web Services Behind Cloudflare"
date: 2023-07-15T21:57:40+07:00
lastmod: 2023-07-16T14:45:40+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "This article will guide you on how to securely hide your website behind Cloudflare."
aliases:
- /posts/go_solid/
images: []
featuredImage: "featured-image.jpg"
featuredImagePreview: "featured-image.jpg"

tags: ["Cloudflare", "DevOps", "DevOpsSec"]
categories: ["DevOps"]

lightgallery: true
---

<!--more-->

## What is Cloudflare?

Cloudflare is a global network of servers. When we add our application to Cloudflare, we use this network to act as an intermediary between requests and our origin server.

Cloudflare sits between requests and our origin server. This position allows us to do many things, such as speeding up content delivery and user experience (CDN), protecting our website from malicious activity (DDoS, firewall), routing traffic (load balancing, waiting rooms), and more.

## How does Cloudflare work?

Using Cloudflare means our domain or subdomain is using proxied DNS records. DNS lookups for our application's URL will resolve to a Cloudflare Anycast IP instead of the original DNS target.

### Before using Cloudflare proxy

{{< mermaid >}}
flowchart LR
  user(Users)
  srv(Servers)
    user <-- http request --> srv
{{< /mermaid >}}

### After using Cloudflare proxy

{{< mermaid >}}
flowchart LR
  user(Users)
  cf(Cloudflare)
  srv(Servers)
    user <-- http request --> cf
    cf <-- http request --> srv
{{< /mermaid >}}

## Seriously Hiding Web Services Behind Cloudflare

From the image above, it seems like our website is safe from the outside world, right? But wait, if you look closely, you'll find that if someone already knew our server's IP address, our website could still be attacked, right? So, the simplest way is to allow only Cloudflare to directly access our web services. We can achieve this using the operating system's firewall.

{{< mermaid >}}
flowchart LR
  user(Users)
  cf(Cloudflare)
  srv(Servers)
    user <-- http request --> cf
    user -- http request --x srv
    cf <-- http request --> srv
{{< /mermaid >}}

## Allow only Cloudflare to access your website

I will use Debian 12 as an example.
> You need `root` privileges.

### Install ipset and its friends
```bash
# install ipset
apt -y install ipset iptables-persistent ipset-persistent curl
```

### Add Cloudflare IPs to ipset
```bash
# get cloudflare IPs
mkdir -p /etc/zones/
curl -L -s https://www.cloudflare.com/ips-v4 -o /etc/zones/cf-ips-v4
curl -L -s https://www.cloudflare.com/ips-v6 -o /etc/zones/cf-ips-v6

# add cloudflare IPs to ipset
ipset -N cloudflare-ips-v4 hash:net
for i in $(cat /etc/zones/cf-ips-v4 ); do ipset -A cloudflare-ips-v4 $i; done
ipset -N cloudflare-ips-v6 hash:net family inet6
for i in $(cat /etc/zones/cf-ips-v6 ); do ipset -A cloudflare-ips-v6 $i; done
```

### Allow http, https only from Cloudflare
```bash
# allow only http/https from cloudflare ipv4
iptables -I INPUT -p tcp -m set --match-set cloudflare-ips-v4 src -m multiport --dports 80,443,8080,8443 -j ACCEPT
iptables -I INPUT -p udp -m set --match-set cloudflare-ips-v4 src -m multiport --dports 80,443,8080,8443 -j ACCEPT
iptables -A INPUT -p tcp -m multiport --dports 80,443,8080,8443 -j DROP
iptables -A INPUT -p udp -m multiport --dports 80,443,8080,8443 -j DROP

# allow only http/https from cloudflare ipv6
ip6tables -I INPUT -p tcp -m set --match-set cloudflare-ips-v6 src -m multiport --dports 80,443,8080,8443 -j ACCEPT
ip6tables -I INPUT -p udp -m set --match-set cloudflare-ips-v6 src -m multiport --dports 80,443,8080,8443 -j ACCEPT
ip6tables -A INPUT -p tcp -m multiport --dports 80,443,8080,8443 -j DROP
ip6tables -A INPUT -p udp -m multiport --dports 80,443,8080,8443 -j DROP
```

### Save
```bash
# save ipset configurations to /etc/iptables/ipsets
dpkg-reconfigure ipset-persistent

# save iptables and ip6tables configurations to /etc/iptables/rules.v{4|6}
dpkg-reconfigure iptables-persistent
```

## Test
```bash
curl -L -v -H "Host: your.domain" your.server.ip.address
```

## Bash script `behind_cloudflare.sh`
{{< gist bouroo 6877eb823c2eee859e50ffeef67ea19e behind_cloudflare.sh >}}
