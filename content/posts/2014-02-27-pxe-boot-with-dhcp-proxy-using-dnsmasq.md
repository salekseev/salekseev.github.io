---
layout: post
title: "PXE boot with DHCP-Proxy using DNSMASQ"
date: 2014-02-27 11:04:00 -0500
comments: true
categories: [linux, pxe, dhcp, dnsmasq, cobbler]
---

The other day I was setting up a cobbler boot server in a network where I
have no control over DHCP server. DHCP-Proxy to the rescue!

I believe the minimum version of DNSMASQ required to function properly
is at least 2.51 as it did not work with out of the box CentOS6 supplied
version 2.48, I ended up using dnsmasq-2.65-5.el6.x86_64.rpm built by
Mirantis.

Here's my working /etc/cobbler/dnsmasq.template file:

```
# Cobbler generated configuration file for dnsmasq
# $date
#

no-poll
enable-dbus
# Disables DNS functionality
port=0
log-dhcp
read-ethers

dhcp-range=172.24.220.0,proxy
dhcp-boot=pxelinux.0
dhcp-ignore=tag:!known
pxe-service=x86PC, "Boot PXELinux (=Cobbler controlled)", pxelinux, $next_server

$insert_cobbler_system_definitions
```
