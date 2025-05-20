---
layout: post
title:  "Protecting QNAP NAS using Cloudflare Proxy and Traefik"
date:   2025-05-09
categories: qnap cloudflare traefik
---
This post outlines how to secure your QNAP NAS using Cloudflare Proxy and an additional reverse
proxy Traefik.

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
security features later on (SSO, CrowdSec). The reverse-proxy then forwards the request to the
target host, the QNAP NAS.

Each of the self-hosted services sits in a dedicated VLAN with strict firewall rules only allowing
traffic between the services and between the designated ports.

# Hardware Prerequisites
Besides the QNAP NAS, we will need a server (e.g. RaspberryPi) running docker. The server's SSH
port must not be accessible from the internet or the vlan your exposed services are running in. 

# Traefik
We set up [Traefik](https://traefik.io/traefik/) with a file provider.

`docker-compose.yml`
```
services:
  traefik:
    image: "traefik:v3.3"
    container_name: "traefik"
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--entryPoints.web.address=:80"
      - "--entryPoints.websecure.address=:443"
      - "--providers.file.directory=/etc/traefik/routes"
      - "--providers.file.watch=true"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    networks:
      vlan2:
        ipv4_address: 192.168.20.2
    volumes:
      - "./traefik/routes:/etc/traefik/routes:r"
networks:
  vlan2:
    driver: ipvlan
    driver_opts:
      parent: eth0.2
    ipam:
      config:
        - subnet: 192.168.20.0/24
          gateway: 192.168.20.1
```
This config sets up Traefik to listen on port 80 and 443. It further exposes Traefik's  management UI on port 8080.

The container is exposed to the `vlan2` network, and receives a static IP. 

The route setup is defined via a file provider, loading config from `/etc/traefik/routes` which we mapped to `./traefik/routes`. We need to create this file.

`./traefik/routes/routes.yml`
```
http:
  # Add the router
  routers:
    router0:
      entryPoints:
      - websecure
      service: nas
      rule: Host(`<your_domain>`)
      tls:
        certResolver: <your-cert-resolver>
  # Add the service
  services:
    nas:
      loadBalancer:
        serversTransport: nas
        servers:
        - url: https://192.168.30.2/
        passHostHeader: false

  serversTransports:
    nas:
      serverName: "<your_nas>.myqnapcloud.com"
```
This specifies a route for host `<your_domain>` (e.g. `nas.example.com`) listening on port `websecure`(443), forwarding this traffic to service `nas`. 
The service `nas` then specifies that the QNAP NAS can be reached via `nas_ip`. 

`serversTransports.nas.serverName` specifies the hostname on the SSL certificate returned by your QNAP NAS. I've mine still set up to pull a cert with a hostname for `myqnapcloud.com`, so needed to set this option.

> [!CAUTION]
> You need to setup Traefik to use SSL/TLS certificates. Please follow the [official documentation](https://doc.traefik.io/traefik/https/overview/) and set the `certResolver` accordingly.

I appreciate that I did not go into much detail on each config option here. Instead I'd like to encourage you to get familiar with the [Traefik Documentation](https://doc.traefik.io/traefik/routing/overview/)

## Adjust firewall to allow traffic between Traefik and your NAS
Adjust your firewall rules so you allow traffic from VLAN2 192.168.20.2 to VLAN3 192.168.30.2 Port 443. No other traffic between the VLAN2 and VLAN3 should be allowed.

## Test connection
Set a DNS entry for your nas (e.g. `nas.example.com`) pointing to Traefik's IP 192.168.20.2 in your local DNS server. Now you can hit the URL in your browser and you should see your NAS' landing page. 

If this is not working, check the Traefik logs via `docker compose logs traefik`. If the logs do not show any traffic or error, then verify your local DNS is setup correctly using `dig nas.example.com`. Check your firewall rules. You should be able to access VLAN2 from your dev machine. Check firewall logs for any denied connections. 

# Expose Traefik to the internet
> [!CAUTION]
> This step exposes Traefik and your NAS to the internet. Anyone can access and try to attack your server. Be aware of the risk. Only continue if you have SSL/TLS setup correctly and followed security best practices to lock down your NAS.

Go to your firewall and setup port forwarding of TCP port 443 to our Traefik server at VLAN2 192.168.20.2. Do not forward any other port.

You should now be able to hit your public IP via the browser and it should resolve to a default page from Traefik (it's some basic reply).

You can use curl `curl --header 'Host: nas.example.com' https://<your_public_ip>` to send the Host header to Traefik, which should forward this request to the NAS.

# Cloudflare proxy
You can follow [Cloudflare's documentation](https://developers.cloudflare.com/fundamentals/setup/manage-domains/add-site/) to add your site/domain to cloudflare.

> [!INFO]
> If your NAS is not hosted behind a static firewall, you need to set up [dynamic DNS](https://www.cloudflare.com/en-gb/learning/dns/glossary/dynamic-dns/). 

In Cloudflare, setup a DNS record pointing at your server's IP. You should enable `Proxy-Mode` which means that all traffic will be routed through a reverse proxy provided by Cloudflare for extra security.

## Adjust the SSL settings 
We need to specify how the Cloudflare proxy connects to our server. In my case, Cloudflare defaulted to `Flexible` which meant that all connections between Cloudflare and my server (origin) were unencrypted. This might be accessible if we are accessing our server via Cloudflare Tunnel, but in our case we are accessing our server via the internet, hence we need to encrypt this traffic.

In Cloudflare, go to `SSL/TLS -> Overview -> Configure` and set the Custom SSL/TLS option based on your needs. I recommend one of the options `Strict, Full (Strict) or Full`.

## Test connection
Now everything should be setup so you can reach your NAS via Cloudflare Proxy.

Here are some trouble shooting steps:

Use `dig <domain>` to verify what IP your domain resolves to. DNS settings might take some time to propagate, so if the IP is wrong, be patient. The IP should point to one of Cloudflare's proxy servers. 

To verify if the DNS record is set correctly (and resolves), you can disable `Proxy-Mode` temporarily. This should send traffic directly to your server.

If you get an HTTP 522 it means that Cloudflare Proxy timed out trying to connect to your server. Check Traefik logs, if they do not show anything then no traffic is hitting the reverse proxy. Check Cloudflare's SSL settings (does it try to connect via HTTP/Port 80 and you only allow access via HTTPS/Port 443?)

# Only allow traffic from Cloudflare Proxy
Once you can connect to your NAS via the Cloudflare Proxy, adjust your firewall rules so it only allows incoming traffic from Cloudflare. See [Cloudflare IP Ranges](https://www.cloudflare.com/en-gb/ips/) for the list. 

## Test firewall rules
Verify that you can still access your NAS via the set domain name.

Hit your public IP address directly and verify that it is not accessible. You can also use services like [dnschecker.org](https://dnschecker.org/port-scanner.php) to do a port scan. Observer your firewall logs. They should show the blocked access attempts. 

# Done
Nice one, that's it!

To further strengthen your security you can look into adding extra services like CrowdSec or Single Sign On to Traefik. You should also run Traefik on a rootless docker engine. 

Lastly, don't forget to regularly update your server and Traefik to avoid running an old, potentially vulnerable software version.


