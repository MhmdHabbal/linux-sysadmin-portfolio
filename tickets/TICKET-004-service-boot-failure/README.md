# TICKET-004: Service Fails on Boot

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/14/2026  
**Skills practiced:** systemd, journalctl, incident response

## 1. Problem Statement
Nginx failed to start after a server reboot on web-01.

## 2. Investigation
Checked the service status immediately after reboot: `$systemctl status nginx`

<pre>
○ nginx.service
    Loaded: masked (Reason: Unit nginx.service is masked.)
    Active: inactive (dead)
</pre>

Checked systemd's journal for this unit: `$sudo journalctl -u nginx --since "10 min ago"`

<pre>
-- No entries --
</pre>

## 3. Root Cause
The service had been masked, which blocks it from starting even via
`systemctl start` or as a dependency of another service — a stronger and
less obvious failure mode than a simple "disabled" state.

## 4. Fix
`$sudo systemctl unmask nginx`

<pre>
Removed '/etc/systemd/system/nginx.service'.
</pre>

`$sudo systemctl enable --now nginx`

<pre>
Synchronizing state of nginx.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable nginx
</pre>

## 5. Verification
`$systemctl status nginx`

<pre>
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-10-14 19:58:56 EEST; 44s ago
 Invocation: 100b372abe1e49bb9267e2260480a0be
       Docs: man:nginx(8)
   Main PID: 4647 (nginx)
      Tasks: 3 (limit: 1856)
     Memory: 4.6M (peak: 5.1M)
        CPU: 120ms
     CGroup: /system.slice/nginx.service
             ├─4647 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─4648 "nginx: worker process"
             └─4649 "nginx: worker process"

Oct 14 19:58:56 web-01 systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Oct 14 19:58:56 web-01 systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.
</pre>

## 6. Prevention
Documented that `systemctl status` output should always be checked
specifically for "masked," since it looks different from a normal
"disabled" state and is easy to miss at a glance.

## 7. What I'd do differently in production
I'd add a boot-time health check or monitoring alert so a masked critical
service is caught within minutes of a reboot, not at the next manual
check.
