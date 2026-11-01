# TICKET-012: Monitoring Stack (Prometheus + Grafana)

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/1/2026  
**Skills practiced:** Prometheus, Grafana, monitoring, VM

## 1. Problem Statement
No visibility existed into server health — issues like TICKET-001's disk
fill-up were only caught after they became critical, not before.

## 2. Investigation
N/A — build ticket rather than a reactive incident.

## 3. Root Cause
No monitoring or alerting existed on web-01 prior to this.

## 4. Fix
Installed node_exporter and Prometheus as systemd services and Grafana from
its apt repository; configured Prometheus to scrape web-01's metrics on
port 9100 and added a Grafana panel + alert rule for disk usage crossing 80%
(query used: the PromQL line from the guide):  

The alert rule configuration on Grafana (Done on the Grafana 'localhost'):
<pre>
100 - (node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs|overlay"} * 100)
</pre>

## 5. Verification
Re-ran the disk-fill scenario from TICKET-001 and confirmed the alert
fired:  

Before the scenario incident: `$df -h /`
<pre>
Filesystem      Size  Used Avail Use% Mounted on 
/dev/sda2        25G   15G  8.6G  64% /
</pre>

<img width="960" height="355" alt="Screenshot 2026-11-01 183626" src="https://github.com/user-attachments/assets/08ae260c-0e6d-4af9-888e-84ebd0856d4e" />


Similar scenario to TICKET-001:
<pre>
$SIZE=$(df --output=size -B1 / | tail -1 | tr -d ' ')
$USED=$(df --output=used -B1 / | tail -1 | tr -d ' ')
$NEED=$(( SIZE * 85 / 100 - USED ))
$echo "Will create $NEED bytes"
$sudo fallocate -l "$NEED" /fill.img
$df -h /
</pre>

And after the scenario: `$df -h /`

<pre>
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        25G   21G  2.5G  90% /
</pre>

Got an alert on Grafana as it should:

<img width="1005" height="333" alt="Screenshot 2026-11-01 181707" src="https://github.com/user-attachments/assets/025085b9-72f8-43ec-b394-7b7618ad8702" />
<img width="961" height="368" alt="Screenshot 2026-11-01 181729" src="https://github.com/user-attachments/assets/84f96a50-6637-4565-bdc6-86dcee232f00" />


## 6. Prevention
This closes the exact gap that let TICKET-001 go unnoticed until the disk
was already full.

## 7. What I'd do differently in production
I'd run this monitoring stack on a separate host from the server being
monitored, so an outage on web-01 doesn't also take down the alerting that
should report it.
