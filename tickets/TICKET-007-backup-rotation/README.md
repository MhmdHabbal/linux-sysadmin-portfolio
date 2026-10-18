# TICKET-007: Backup Rotation

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/18/2026  
**Skills practiced:** bash scripting, automated rotation policies

## 1. Problem Statement
Backups were accumulating indefinitely with no cleanup, risking the same
disk fill-up failure mode documented in TICKET-001.

## 2. Investigation
N/A — this was a proactive improvement rather than a reactive incident,
made after recognizing the same risk pattern as TICKET-001.

## 3. Root Cause
No retention policy existed in the original backup script from TICKET-002.

## 4. Fix
Added an automatic cleanup line to the existing backup script:

<pre>
find /backup -name "*.tar.gz" -mtime +7 -delete
</pre>

## 5. Verification
Simulated an old backup file and confirmed the script cleaned it up while
keeping the current one: `$touch -d "10 days ago" /backup/etc-backup-old-test.tar.gz`  


After adding the <backup-old-test> file and running: `$ls -lh /backup`

<pre>
total 1.3M
-rw-r--r-- 1 root root 1.3M Oct 18 12:08 etc-backup-2026-10-18.tar.gz
-rw-r--r-- 1 root root    0 Oct  8 12:26 etc-backup-old-test.tar.gz
</pre>

After adding the file and running the script `$sudo /opt/scripts/backup.sh`: `$ls -lh /backup`

<pre>
total 1.3M
-rw-r--r-- 1 root root 1.3M Oct 18 12:28 etc-backup-2026-10-18.tar.gz
</pre>

*The file was deleted as intended*

## 6. Prevention
Retention is now baked into the script itself instead of relying on a
human to clean up manually.

## 7. What I'd do differently in production
I'd also verify a separate off-site/off-server copy exists before ever
auto-deleting local backups, so a bug in the retention logic can't destroy
the only copy.
