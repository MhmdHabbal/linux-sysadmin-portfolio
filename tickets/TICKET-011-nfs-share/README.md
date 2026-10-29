# TICKET-011: NFS File Share

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/29/2026  
**Skills practiced:** NFS, file sharing, Linux networking

## 1. Problem Statement
Needed a shared file location accessible from multiple machines on the lab
network.

## 2. Investigation
N/A — build/setup ticket rather than a reactive incident.

## 3. Root Cause
N/A — describing a requirement, not a failure.

## 4. Fix
Installed the NFS server on web-01, exported `/srv/shared` scoped to
client-01's IP, and mounted it from client-01:

<pre>
$sudo apt install -y nfs-kernel-server
$sudo mkdir -p /srv/shared
$echo "/srv/shared <client-01-IP>(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
$sudo exportfs -a
$sudo systemctl restart nfs-kernel-server
</pre>

## 5. Verification

Output of `$df -h | grep shared` from client-01:

<pre>
<web-01-IP>:/srv/shared   25G   12G   12G  50% /mnt/shared
</pre>

I've created a test folder on client_01 and I've confirmed that it is appearing on web-01(server).

## 6. Prevention / Notes
Scoped the export to a specific client IP rather than opening it to the
whole subnet.

## 7. What I'd do differently in production
I'd add proper user/permission mapping (considering `no_root_squash`
implications) and look at Kerberos-secured NFS for anything beyond a
trusted lab network.
