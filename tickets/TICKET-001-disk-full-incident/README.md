# TICKET-001: Disk Full Incident on web-01

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/4/2026  
**Skills practiced:** disk troubleshooting, logrotate, incident response

## 1. Problem Statement
Simulated monitoring alert: disk usage on web-01 exceeded 90% on the root
partition. Investigated as if responding to a live page.

## 2. Investigation
Checked overall disk usage: `$df -h`

<pre>
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           680M  2.1M  678M   1% /run
/dev/sda2        25G   13G   12G  53% /
tmpfs           1.7G     0  1.7G   0% /dev/shm
none            1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
tmpfs           1.7G  8.0K  1.7G   1% /tmp
none            1.0M     0  1.0M   0% /run/credentials/systemd-resolved.service
tmpfs           340M   80K  340M   1% /run/user/1000
</pre>

Found the largest consumers under /var/log: `$du -sh /var/log/* 2>/dev/null | sort -rh | head -10`

<pre>
1001M   /var/log/fakeapp
149M    /var/log/journal
5.3M    /var/log/syslog
4.2M    /var/log/syslog.1
1.5M    /var/log/installer
1.2M    /var/log/kern.log
1.1M    /var/log/dpkg.log.1
964K    /var/log/kern.log.1
380K    /var/log/boot.log.2
264K    /var/log/boot.log.1
</pre>

Identified `/var/log/fakeapp/` as the source — hundreds of uncontrolled
log files with no rotation policy.

## 3. Root Cause
An application (`fakeapp`) was writing log files with no rotation or size
cap configured, eventually consuming all free disk space.

## 4. Fix
Removed the accumulated log files after confirming they were not needed: `$sudo rm -rf /var/log/fakeapp`

## 5. Verification
Checked the disk usage: `$df -h /`

<pre>
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        25G   12G   13G  49% /
</pre>

## 6. Prevention
Added a logrotate policy for this app's log directory (daily rotation,
7-day retention, compression, 50MB size cap):

```
$sudo tee /etc/logrotate.d/fakeapp <<'EOF'
/var/log/fakeapp/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    maxsize 50M
}
EOF
```

Verified the policy with a dry run: `$sudo logrotate -d /etc/logrotate.d/fakeapp`

<pre>
warning: logrotate in debug mode does nothing except printing debug messages!  Consider using verbose mode (-v) instead if this is not what you want.
reading config file /etc/logrotate.d/fakeapp
Reading state from file: /var/lib/logrotate/status
Allocating hash table for state file, size 64 entries
Creating new state
Creating new state
Creating new state
Creating new state
.
.
.
.
.
etc.

Handling 1 logs

rotating pattern: /var/log/fakeapp/*.log after 1 days empty log files are not rotated, log files >= 52428800 are rotated earlier, (7 rotations), old logs are removed
considering log /var/log/fakeapp/*.log
Creating new state
</pre>

## 7. What I'd do differently in production
I'd check with the app owner before deleting any logs rather than assuming they're safe to
remove.
