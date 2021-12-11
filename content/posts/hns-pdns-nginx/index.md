---
title: 'HNS + PowerDNS + Nginx + DANE - Introduction'
date: 2021-11-26T00:00:00+05:30
draft: false
---

**Posts in this series:**

1. Introduction (this one!)
2. [Part 1 - Set up DNS server]({{< ref "hns-pdns-nginx-part-1" >}})
3. [Part 2 - Set up Web server]({{< ref "hns-pdns-nginx-part-2" >}})
4. [Part 3 - Secure with DANE]({{< ref "hns-pdns-nginx-part-3" >}})

---

So you wanna build a ~~snowman~~ website on a Handshake domain.

Any website (even a regular one on an ICANN domain) needs 3 things to function

1. A domain name - hope you have one ready
2. A DNS server - this is what tells browsers where to fetch content from
3. A web server - someplace where content is stored or served from

Let's quickly go through what they are (feel free to skip if you're familiar
with DNS basics).

### Domain Name

Similar to how you'd ~~buy~~ rent a regular domain from a registrar like
Namecheap or GoDaddy, get a domain on Handshake from https://namebase.io or with
a non-custodial wallet like [Bob Wallet](https://bobwallet.io).

### DNS Server

An authoritative DNS nameserver (We'll call this a DNS server from here on) is
like a redirector that stores all the subdomains (and their records) and
pointers to where content is hosted. Registrars normally provide this service
for free for domains you manage with them. In this series, we'll set up our own
(with PowerDNS).

### Web Server

A web server hosts the content that should be shown on the website. GitHub
pages, Vercel, Netlify, etc. all do this, but again in this series, we'll set
up our own (with nginx).

### How do these 3 things work together?

1. Domain name points to DNS server
2. DNS server has records that points to Web server
3. Web server has files and content to serve

## Get a cloud Virtual Machine

We'll have all this running on a single (cloud) machine, but they can be split
up if you wish.

Most low-traffic sites can do away with a small $5/mo cloud VM from [Digital
Ocean](https://www.digitalocean.com/products/droplets/) (or any other cloud).
This series uses a `B1ls` size on
[Azure](https://azure.microsoft.com/en-us/services/virtual-machines/) with 1
vCPU and 0.5 GiB of memory. Feel free to scale up if the website starts getting
more traffic.

While creating a VM, make sure to open 3 ports:

- 53 for DNS
- 80 and 443 for HTTP and HTTPS

> **Note:** This series does not cover taking backups, setting up monitoring, etc. that
> you'd want to do with a typical production-level site. There's nothing
> different about handshake websites and guides online work as-is for all those
> things.

This series uses these values as an example. Replace them with your own in all
commands:

- Domain: `smartface`
- IP address of machine: `20.106.52.247`

For people who have a single and simple website and don't plan to add more
websites, there's a handy software package that wraps DNS and Web server in a
single app, and takes care of DANE. You may want to use that instead:
https://github.com/pinheadmz/handout

This whole post is divided into 3 parts and in each one, there are alternatives
mentioned if you prefer other software.

Before we start, credit where due (in no specific order):

- [pinheadmz](https://github.com/pinheadmz) for [this
  article](https://matthewzipkin.medium.com/building-a-secure-website-on-your-handshake-tld-a8922a950a4f)
  (along with others) and [handout](https://github.com/pinheadmz/handout)
- [Buffrr](https://github.com/buffrr) for [Let's
  DANE](https://github.com/buffrr/letsdane) and
  [Fingertip](https://github.com/imperviousinc/fingertip) that make DANE easy to
  use
- [A guide to hosting static websites using NGINX](https://jgefroh.medium.com/a-guide-to-using-nginx-for-static-websites-d96a9d034940)
- [PowerDNS](https://doc.powerdns.com/authoritative/index.html)

To start, check out [Part 1: Set up the DNS server]({{< ref
"hns-pdns-nginx-part-1" >}}).
