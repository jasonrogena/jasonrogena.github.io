---
title: Running My Home Lab Using DNS-SD
date: 2026-07-12
tags: [DNS-SD, DNS, Caddy]
---

> This post has NOT been generated using AI.

Like most of those who I believe would be reading this blog, I run my own home lab. My setup has, over the years, evolved from an old laptop running just my media centre, only accessible on my home network, to a more elaborate setup with containers running on two hosts on my local network, a third host running local LLMs, and a fourth (thin) host in the cloud running a reverse proxy. The reverse proxy forwards requests to services running in the hosts on my local network, allowing me to access some services from anywhere. I like the setup. In essence, it is quite simple; all I have to deal with is a container engine (I use Podman Quadlet) and my reverse proxy, outside the actual services deployed in the lab. I have avoided jumping into the Kubernetes (or similar) bandwagon, as I still don't think the overhead is justified as yet.

It being a home lab means I quite often bring up and take down services. One big pain point for me was having to SSH into my cloud host to update the reverse-proxy configuration every time I'd want to expose or stop exposing a service to the internet. So last year, while in between jobs, I started exploring how I could auto-configure which services to expose to the internet. I was interested in mechanisms that would be agnostic to whatever container engine I use (having moved from systemd-nspawn to Podman, and afraid I'd do a similar switch soon after).

## SRV for the Win

DNS was coming out on top for this kind of container engine agnostic mechanism of service discovery. The rough idea was I would configure my reverse proxy to accept requests from any subdomain in a wildcard domain (let's say `*.apps.rogena.me`). The reverse proxy would extract whatever subdomain was being hit, let's say `home-assistant` in `home-assistant.apps.rogena.me`, and forward the request to a service running inside whatever container IP address either `home-assistant.host1.lan` or `home-assistant.host2.lan` resolves to.

Two things I had to figure out:

- Typical DNS record types (A and AAAA) would only allow discovering the IP address for the container, and not the port the service is running on in the container.
- What would load-balancing across multiple containers exposing the same service look like?

SRV is a lesser-known DNS record type that allows you to discover both the IP address and port a service is running on. Additionally, you can encode, as part of an SRV record, the priority and weight to use for the returned host-port pairs when trying to decide which one to use if multiple host-port pairs are returned when a query is made for an SRV record. The priority field encodes "who to send the request to first". A lower value for this field indicates a higher priority. Under normal operation, requests should only be sent to the host-port pair with the lowest priority value. The weight field encodes how to split load amongst the host-port pairs that have the lowest priority value. A higher weight indicates a host-port pair can take more load. With these two fields, you can encode your fail-over as well as load-balancing strategy.

```console
# dig home-assistant._http._tcp.host1.lan. SRV

; <<>> DiG 9.18.39-0ubuntu0.22.04.4-Ubuntu <<>> home-assistant._http._tcp.host1.lan. SRV
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48215
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;home-assistant._http._tcp.host1.lan. IN SRV

;; ANSWER SECTION:
home-assistant._http._tcp.host1.lan. 60 IN SRV 0 77 8123 0.home-assistant.host1.lan.

;; ADDITIONAL SECTION:
0.home-assistant.host1.lan. 60 IN A 192.168.10.58

;; Query time: 44 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Sun Jul 12 20:11:31 UTC 2026
;; MSG SIZE  rcvd: 169
```

The above shows an example dig for an SRV record `home-assistant._http._tcp.host1.lan.`. `_http` and `_tcp` in this example encode the service name and protocol. In the answer, `0 77` shows the priority and weight, while `8123 0.home-assistant.host1.lan.` shows the port and the A record with the IP address of the container running home-assistant.

Great success! I'm able to discover which container IP and port I should hit to access a service. The reverse proxy can, in theory, run a DNS SRV query for `home-assistant._http._tcp.host1.lan.` and `home-assistant._http._tcp.host2.lan.` when a user tries accessing `https://home-assistant.apps.rogena.me`. In a section below, I explain exactly how I configured Caddy (my reverse proxy).

I couldn't find a container engine agnostic DNS server that exposes SRV records for containers running on hosts, so I wrote [container-dns](https://github.com/jasonrogena/container-dns). container-dns runs inside host1 and host2. With container-dns, all I need to do is to configure an appropriate hostname for the container and add the http port for the service in the container's `/etc/services` file. With `home-assistant._http._tcp.host1.lan.`, I set the hostname for the container as `home-assistant` and add a row in `/etc/services` in the container with `http 8123/tcp` (the name of the service and the TCP port the service is running on).

## Discovering Extra Service Metadata

More recently, I have explored using DNS to solve a different discovery pain-point with my home lab. I wanted to auto-enable uptime monitoring for all the services I expose to the internet. Naturally, some services running in my home lab are now being used by people other than me. I'd love to know when these services are down before they do.

I explored how to auto-discover:
- Which services to do uptime monitoring for
- Which domain names these services are running under
- What path in the domain is most appropriate to monitor
- What HTTP status code and content I should expect to be returned if a service is healthy

Turns out the SRV record mechanism container-dns supports is part of a larger specification called [DNS-SD](https://www.rfc-editor.org/info/rfc6763/). DNS-SD also specifies how to enumerate which services are deployed under a domain as well as extra metadata for a service. I would use these to auto-discover which services to do uptime monitoring for, as well as how to monitor them.

I'll attempt to explain how I do this using a chain of DNS queries. To discover all services running in host1, you would query a PTR record (`_services._dns-sd._udp` is documented in the DNS-SD spec):

```console
# dig _services._dns-sd._udp.host1.lan PTR
;; Truncated, retrying in TCP mode.

; <<>> DiG 9.18.39-0ubuntu0.22.04.4-Ubuntu <<>> _services._dns-sd._udp.host1.lan PTR
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5303
;; flags: qr rd ra; QUERY: 1, ANSWER: 50, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;_services._dns-sd._udp.host1.lan. IN PTR

;; ANSWER SECTION:
_services._dns-sd._udp.host1.lan. 59 IN PTR _http._tcp.host1.lan.
_services._dns-sd._udp.host1.lan. 59 IN PTR _ssh._tcp.host1.lan.
_services._dns-sd._udp.host1.lan. 59 IN PTR _postgresql._tcp.host1.lan.
```

Looks like there are `http`, `ssh`, and `postgresql`. For `http`, you can do a PTR query to check which instances expose the service by querying the PTR record:

```console
# dig _http._tcp.host1.lan PTR

; <<>> DiG 9.18.39-0ubuntu0.22.04.4-Ubuntu <<>> _http._tcp.host1.lan PTR
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 27672
;; flags: qr rd ra; QUERY: 1, ANSWER: 15, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;_http._tcp.host1.lan. IN  PTR

;; ANSWER SECTION:
_http._tcp.host1.lan. 60 IN PTR    yarr._http._tcp.host1.lan.
_http._tcp.host1.lan. 60 IN PTR    home-assistant._http._tcp.host1.lan.
_http._tcp.host1.lan. 60 IN PTR    grafana._http._tcp.host1.lan.

;; Query time: 44 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Sun Jul 12 21:28:25 UTC 2026
;; MSG SIZE  rcvd: 396
```

Looks like an http service is available in yarr, home-assistant, and grafana. Note how the record `home-assistant._http._tcp.host1.lan.` matches the SRV record we covered above. We can also get the metadata for the home-assistant http service by querying a TXT record (corresponding to the SRV record for the service):

```console
# dig  home-assistant._http._tcp.host1.lan. TXT

; <<>> DiG 9.18.39-0ubuntu0.22.04.4-Ubuntu <<>> home-assistant._http._tcp.host1.lan. TXT
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40093
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;home-assistant._http._tcp.host1.lan. IN TXT

;; ANSWER SECTION:
home-assistant._http._tcp.host1.lan. 60 IN TXT "domain=home-assistant.apps.rogena.me" "healthz=/auth/authorize" "okstatus=200"

;; Query time: 44 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Sun Jul 12 21:33:00 UTC 2026
;; MSG SIZE  rcvd: 163
```

The answer for the TXT record contains several metadata fields: domain, healthz, and okstatus. I can use these to configure how to do uptime monitoring for Home Assistant (i.e., attempt to reach "https://home-assistant.apps.rogena.me/auth/authorize" and check if it returns the HTTP status code 200).

I extended container-dns to check in each of the containers for the `/etc/container-dns/txt` file and expose the data in that file in the TXT record for the services the container runs. For Home Assistant, this is how the file looks:

```ini
[8123/tcp]
domain=home-assistant.apps.rogena.me
healthz=/auth/authorize
okstatus=200
```

I wrote [upwatch](https://github.com/jasonrogena/upwatch), an uptime monitoring service that talks DNS-SD. I don't need to directly configure it to add or remove a monitor for a service. I just have to make sure the `/etc/container-dns/txt` file in the container running the service has the right fields configuring how to monitor the service.


## Configuring Caddy

I use Caddy as my reverse proxy exposing the services I run in my home lab to the internet. This is how my Caddyfile looks:


```
*.apps.rogena.me {
    tls {
        dns cloudflare <REDACTED>
    }
    reverse_proxy {
        dynamic multi {
            srv {labels.3}._http._tcp.host1.lan {
                refresh 15s
                grace_period 2m
            },
            srv {labels.3}._http._tcp.host2.lan {
                refresh 15s
                grace_period 2m
            }
        }
    }
}
```

Caddy calls the DNS resolver on the host it is running on. I have configured the resolver (systemd-resolved) to forward DNS queries for host1 and host2 to the corresponding instance of container-dns.
