# TICKET-000: SSH Connection Refused on web-01

**Status:** Resolved
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, connecting from a WSL (Ubuntu) host on Windows
**Date:** 9/27/2026
**Skills practiced:** SSH setup, systemd, real-world troubleshooting (unscripted)

## 1. Problem Statement
While setting up remote access to web-01 for the first time, connecting
from my WSL host failed with:
`ssh: connect to host 192.168.56.103 port 22: Connection refused`

## 2. Investigation
"Connection refused" (rather than a timeout) indicated the VM was
reachable on the network but nothing was accepting connections on port 22.
Since SSH itself wasn't working, I logged into web-01 directly through the
VirtualBox console window and checked the service:
`$ sudo systemctl status ssh`
Result:
`unit not found — SSH server was not installed`

## 3. Root Cause
OpenSSH server had never been installed on web-01 — likely skipped during
the initial Ubuntu Server installation.

## 4. Fix
`$ sudo apt update && sudo apt install -y ssh`

## 5. Verification
Returned to my WSL host and reconnected successfully:
`$ ssh username@<web-01-IP>` [Connected successfully]

## 6. Prevention
Will explicitly check the "Install OpenSSH server" option during future VM
installs, or verify `systemctl status ssh` immediately after first boot,
before assuming remote access is available.

## 7. What I'd do differently in production
I'd bake SSH server installation into a base image or an automated
provisioning playbook (Ansible) rather than relying on a manual checklist
step, so this can't be silently skipped on a new machine.
