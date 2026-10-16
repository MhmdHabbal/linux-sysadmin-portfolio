# TICKET-005: Firewall Blocks Traffic Between Machines

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/16/2026  
**Skills practiced:** networking, ufw, firewall troubleshooting

## 1. Problem Statement
client-01 could not reach the web server on web-01 over HTTP.

## 2. Investigation
Confirmed the failure from client-01: `$curl -m 3 http://<web-01-IP>`

<pre>
curl: (28) Connection timed out after 3003 milliseconds
</pre>

Checked web-01's firewall rules: `$sudo ufw status numbered`

<pre>
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 80                         DENY IN     Anywhere
[ 2] 2222/tcp                   ALLOW IN    Anywhere
[ 3] 80 (v6)                    DENY IN     Anywhere (v6)
[ 4] 2222/tcp (v6)              ALLOW IN    Anywhere (v6)
</pre>

## 3. Root Cause
A firewall rule on web-01 explicitly denied all traffic to port 80.

## 4. Fix
`$sudo ufw delete deny 80`

<pre>
Rule deleted
Rule deleted (v6)
</pre>

`$sudo ufw allow 80/tcp`

<pre>
Rule added
Rule added (v6)
</pre>

## 5. Verification
Re-ran the same test from client-01: `curl -m 3 http://<web-01-IP>`

<pre>
*Showed the HTML code for Nginx Welcome message*
</pre>

## 6. Prevention
Adopted the habit of running `ufw status numbered` before troubleshooting
any "can't connect" issue, since it's the fastest way to spot an
unintended block.

## 7. What I'd do differently in production
I'd check both the host firewall AND any network-level firewall/security
group, since either could cause the identical symptom, and I'd document
every intentional firewall rule with a comment explaining why it exists.
