# TICKET-002: Silent Cron Job Failure

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/11/2026  
**Skills practiced:** cron, bash scripting, error handling, logging

## 1. Problem Statement
A nightly backup cron job (`/opt/scripts/backup.sh`) was reportedly producing
no backups, with no error visible anywhere.

## 2. Investigation
The script:

<pre>
#!/bin/bash
tar -czf /backup/etc-backup-$(date +%F).tar.gz /etc
</pre>

Ran the script manually to reproduce: `$sudo /opt/scripts/backup.sh`

<pre>
/opt/scripts/backup.sh: line 2: data: command not found
tar: Removing leading `/' from member names
tar (child): /backup/etc-backup-.tar.gz: Cannot open: No such file or directory
tar (child): Error is not recoverable: exiting now
tar: /backup/etc-backup-.tar.gz: Cannot write: Broken pipe
tar: Child returned status 2
tar: Error is not recoverable: exiting now
</pre>

Confirmed the target directory didn't exist: `$ls -la /backup`

<pre>
ls: cannot access '/backup': No such file or directory
</pre>

Checked cron's own log for related entries: `$grep CRON /var/log/syslog | tail -5`

<pre>
2026-10-11T14:30:01.505297+03:00 web-01 CRON[7845]: (root) CMD (if [ -x /etc/init.d/anacron ] && ! [ -d /run/systemd/system ]; then exec /usr/sbin/invoke-rc.d anacron start >/dev/null; fi)
2026-10-11T15:17:01.620628+03:00 web-01 CRON[8473]: (root) CMD (cd / && run-parts --report /etc/cron.hourly)
2026-10-11T15:30:01.660192+03:00 web-01 CRON[8749]: (root) CMD (if [ -x /etc/init.d/anacron ] && ! [ -d /run/systemd/system ]; then exec /usr/sbin/invoke-rc.d anacron start >/dev/null; fi)
2026-10-11T16:17:01.145850+03:00 web-01 CRON[9068]: (root) CMD (cd / && run-parts --report /etc/cron.hourly)
2026-10-11T16:30:01.206765+03:00 web-01 CRON[9140]: (root) CMD (if [ -x /etc/init.d/anacron ] && ! [ -d /run/systemd/system ]; then exec /usr/sbin/invoke-rc.d anacron start >/dev/null; fi)
</pre>

## 3. Root Cause
The script assumed `/backup` already existed and had no error handling. 
`tar` failed silently (cron doesn't email failures by default on this box), so nobody was ever notified.

## 4. Fix
Rewrote the script with `set -euo pipefail`, explicit directory creation, and explicit success/failure logging to `/var/log/backup.log` as shown below.

<pre>
#!/bin/bash
set -euo pipefail
LOGFILE=/var/log/backup.log
BACKUP_DIR=/backup
mkdir -p "$BACKUP_DIR"
if tar -czf "$BACKUP_DIR/etc-backup-$(date +%F).tar.gz" /etc; then
	echo "$(date): Backup SUCCESS" >> "$LOGFILE"
else
	echo "$(date): Backup FAILED" >> "$LOGFILE"
	exit 1
fi
</pre>

## 5. Verification
Ran the new script [SUCCESSFULY]: `$sudo /opt/scritps/backup.sh`  
`cat /var/log/backup.log`

<pre>
Sun Oct 11 01:57:20 PM EEST 2026: Backup SUCCESS
</pre>

`ls -lh /backup`

<pre>
total 1.2M
-rw-r--r-- 1 root root 1.2M Oct 11 17:55 etc-backup-2026-10-11.tar.gz
</pre>

## 6. Prevention
All future scheduled scripts on this box now log explicitly and exit
non-zero on failure so monitoring can catch it.

## 7. What I'd do differently in production
I'd also wire this into an alerting system (e.g. a healthcheck ping service,
or have cron mail failures to an ops address) rather than relying on someone
noticing a missing log line.

