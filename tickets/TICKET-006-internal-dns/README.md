# TICKET-006: Internal DNS Setup

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/17/2026  
**Skills practiced:** DNS, dnsmasq, internal networking

## 1. Problem Statement
Needed an internal DNS resolver so machines on the lab network can reach
services by name instead of IP.

## 2. Investigation
N/A — this was a build/setup ticket rather than a reactive incident.
Confirmed beforehand that the hostname didn't resolve at all:

<pre>
Server:         127.0.0.1
Address:        127.0.0.1#53

Name:   app.internal
Address: 192.168.56.10
** server can't find app.internal: REFUSED
</pre>

## 3. Root Cause
N/A — no DNS service existed yet on the lab network; this documents the
requirement rather than a failure.

## 4. Fix
Installed dnsmasq and added a static hostname mapping:

<pre>
host-record=app.internal,192.168.56.10
</pre>

The first restart failed with `failed to create listening socket for port 53: Address already in use`, because `systemd-resolved` already holds a
local DNS listener on `127.0.0.53:53` and dnsmasq tries to claim the
wildcard address by default. Confirmed the conflict, then fixed it by
setting `bind-interfaces` plus explicit `listen-address` lines for
loopback and web-01's lab IP, and opened port 53 in `ufw` (enabled in
TICKET-005) so client-01's queries wouldn't be silently dropped:

<pre>
tcp   LISTEN 0      4096       127.0.0.54:53        0.0.0.0:*    users:(("systemd-resolve",pid=5723,fd=20))
</pre>

<pre>
bind-interfaces
listen-address=127.0.0.1
</pre>

## 5. Verification
From web-01: `nslookup app.internal 127.0.0.1`

<pre>
Server:         127.0.0.1
Address:        127.0.0.1#53

Name:   app.internal
Address: <host-record-address>
</pre>

From client-01: `nslookup app.internal <web-01-IP>`

<pre>
Server:         <web-01-IP>
Address:        <web-01-IP>#53

Name:   app.internal
Address: <host-record-address>
</pre>

`nslookup` also printed an extra error line for the IPv6 (AAAA) query,
which my config doesn't define. On a fresh setup this was REFUSED
(dnsmasq had no upstream to forward to); `local=/internal/` changed it to
NXDOMAIN; defining the name with `host-record` produced the precise
NODATA ("No answer") response. None of these affected the IPv4 lookup
the ticket required: `nslookup -type=AAAA app.internal 127.0.0.1`

<pre>
Non-authoritative answer:
*** Can't find app.internal: No answer
</pre>

## 6. Prevention / Notes
Documented the config line format so any new internal hostname can be
added the same way going forward. On any Ubuntu box, check what already
holds a port (`ss -tulnp`) before installing a service that wants it, and
remember that an enabled firewall needs an explicit allow rule for any new
service. A `resolvconf ... dbus-org.freedesktop.network1.service not found`
line in dnsmasq's status is a harmless warning about web-01's own resolver
registration, not a failure of the DNS service itself.

## 7. What I'd do differently in production
A real environment would use a proper DNS server (e.g. BIND) with
redundancy — dnsmasq is fine for a small lab but is a single point of
failure here.
