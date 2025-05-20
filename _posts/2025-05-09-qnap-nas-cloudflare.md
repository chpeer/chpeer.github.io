---
layout: post
title:  "Protecting QNAP NAS using Cloudflare Proxy and Trafik"
date:   2025-05-09
categories: qnap cloudflare trafik
---
This post outlines how to secure your QNAP NAS using Cloudflare Proxy and an additional reverse
proxy Trafik.

> [!CAUTION]
> Exposing your NAS to the internet makes it vulnerable to attacks - only do this if you know
> what you are doing, and at your own risk.

# Background
The widespread recommendation is to not have your QNAP NAS accessible from the internet in order to
protect it from potential attacks. Personally, I struggle with this recommendation. I want an alternative
to the public cloud services, and I want it to be conveniently accessible.

At the time of writing, I've had my NAS exposed for over 7 years without issues. Mind you though,
the NAS sits behind an enterprise grade firewall which was blocking about 1600 attacks on the NAS.
Further, I've had plenty of bots trying to brute force their way in (deactivate the default admin
account).

Anyway, it's time for a change, I'd like to further improve the security of my setup (and get rid of
those annoying bots).

# Target Architecture

< insert diagram>

I'm using Cloudflare as a proxy, which already blocks known bad actors and attacks. Cloudflare then
forwards the traffic through the firewall to the Traefik, which is acting as the internal
reverse-proxy. While this adds another layer of security, it also allows me to add additional
security features later on (SSO, crowdsec). The reverse-proxy then forwards the request to the
target host, the QNAP NAS.

Each of the self-hosted services sits in a dedicated VLAN with strict firewall rules only allowing
traffic between the services and between the designated ports.

# Hardware Preqrequisits
Besides the QNAP NAS, we will need a server (e.g. RaspberryPi) running docker. The server's ssh
port must not be accessible from the internet or the vlan your exposed services are running in.

# Traefik
We set up [Traefik](https://traefik.io/traefik/) with a file provider:

<insert docker compose files>
set up the route/service
ssl certificates
vlan config

adjust firewall to allow connection from traefik to nas

local dns
test if it is working - what if it is not working?

# Expose Traefik to the internet
Set up port forwarding in your firewall
Ensure firewall is set up so you only allow traffic on one port 443 to traefik

test this = default page if no host provided

provide host with curl -> should resolve

set dns -> should resolve


# Cloudflare proxy

Assumption that domain/DNS is managed by cloudflare
enable proxy mode
change SSL settings
test connection
use dig to see what your dns is resolving

if you get a 522 -> cloudflare proxy can't reach your host (timeout) -> check traefik logs, if
nothing shows then your firewall config is wrong, or ssl config is off

once it is working, adjust firewall to only allow incoming traffic from cloudflare ip list <link>

test hitting your service directly via IP (use external service or connection) and verify that you
get bocked (check firewall logs)

you might need to configure dyn dns -> link to cloudflare documentation

