---
title: Converting Hex Strings to IP Addresses with Ruby
author: Heston Snodgrass
date: 2021-02-01 16:45:00 -0700
catagories: [SystemsAdministration]
tags: [haproxy, letsencrypt, proxy, tls, homelab]
math: true
mermaid: true
---

I recently moved from the excellent [Caddy](https://caddyserver.com/v2) to [HAProxy](http://www.haproxy.org/) for my homelab's reverse-proxy.
This change was due to some expanded functionatlity I wanted that Caddy couldn't provide as part of a larger homelab reorginization. I may get around to writing
about that someday, but today I wanted to write about the best feature of Caddy and how I got it working with HAProxy: automatic TLS via [Let's Encrypt](https://letsencrypt.org/).
This walkthrough will get you TLS terminated at HAProxy with a Let's Encrypt certificate, and we'll even automate it so the certificate gets automatically
renewed.

> **Note** I have HAProxy running on Ubuntu 20.04, but the steps should be roughly the same regardless of distro used as long as you have `systemd`.

## Pre-Reqs

To start off, you will need a domain for the Let's Encrypt certs and you will need to get [certbot](https://certbot.eff.org/) running as a `systemd` service. You
will also need HAProxy installed on your machine. I intend to write about how to set these things up in the future, but it's out of scope for this article.

## Configuration

Assuming you have HAProxy installed and the HAProxy config is located at `/etc/haproxy/haproxy.cfg`, the next thing we need to do is to create a symlink
in the HAProxy directory to the live directory for Let's Encrypt: `ln -s /etc/letsencrypt/live /etc/haproxy/certs.d`. This makes it easier to configure
HAProxy to read from this directory. If you have HAProxy running as a user besides `root`, you will need to `chown` the new directory:
`chown <user>:<group> /etc/haproxy/certs.d`.

HAProxy frontends require a combined certificate to provide TLS termination. What this means is that the certificate chain and the private key of the TLS
certificate are required to exist in one file. Fortunately, creating a combined `.pem` is easy:
`cat /etc/haproxy/certs.d/<your domain>/fullchain.pem /etc/haproxy/certs.d/<your domain>/privkey.pem > /etc/haproxy/certs.d/<your domain>/<your domain>.pem`.

Now we need to configure HAProxy to use the combined cert for TLS termination. In your HAProxy config, create / modify a `frontend` for your domain.

```text
frontend <your domain>
    bind <IP of HAProxy server>:443 ssl crt /etc/haproxy/certs.d/<your domain>/<your domain>.pem
    mode http
    ...
```

What we've done is bind port `443` to HAProxy's IP and provided our combined cert for it to use. Now, any backend you put behind that frontend will
present as TLS-encrypted with your Let's Encrypt certificate. In order for this change to take effect, we must restart HAProxy: `systemctl restart haproxy`.

There is one last step we need to perform. Certbot can provide a `systemd` timer service to automatically renew your certs from Let's Encrypt. If you
run `systemctl list-timers | grep certbot` you can find the service name. In my case its `snap.certbot.renew.timer`. If you see the timer service,
you are good to go. Without going too much in-depth, `systemd` timers act like `cron` jobs but typically start other `systemd` services. The certbot
service name can be found with `systemctl list-unit-files | grep certbot.*service`. Once you have the service name, mine is `snap.certbot.renew.service`,
we can create the drop-in file.

Using drop-in files in `systemd` is a way to modify the behavior of services without actually editing the unit files. What we want to do now is create
a drop-in file for the certbot renew service that automatically creates the combined `.pem` file we need. We can do this by running the following
command: `systemctl edit <certbot renew service name>`. This will bring up a blank file in your text editor. In this file, add the following line:

```text
ExecStartPost=/bin/cat /etc/letsencrypt/live/home.techmogoyf.dev/fullchain.pem /etc/letsencrypt/live/home.techmogoyf.dev/privkey.pem > /etc/letsencrypt/live/home.techmogoyf.dev/home.techmogoyf.dev.pem; systemctl restart haproxy
```

What will happen now is that every time the renew service runs, it will create our combined `.pem` and restart HAProxy for us. We now have automatic TLS for our domain via Let's Encrypt and HAProxy.
These steps can also be repeated so you can have multiple frontends all with different certs, if you so choose.
