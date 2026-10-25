# TICKET-010: Database Backup/Restore Drill

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/25/2026  
**Skills practiced:** PostgreSQL administration, backup/restore, disaster recovery

## 1. Problem Statement
Practiced a full database disaster-recovery drill: simulated total loss of
the `appdb` database to confirm backups are actually restorable.

## 2. Investigation
N/A — this was a deliberate drill rather than a reactive incident. The
drill exists because backups that have never been test-restored can't be
trusted to work when actually needed.

## 3. Root Cause
N/A — proactive validation, not a failure response.

## 4. Fix
Dumped the database, deliberately dropped it, recreated an empty database
with the same name, and restored from the dump file:

<pre>
$sudo -u postgres pg_dump appdb > /tmp/appdb_backup.sql
$sudo -u postgres psql -c "DROP DATABASE appdb;"
$sudo -u postgres psql -c "CREATE DATABASE appdb;"
$sudo -u postgres psql appdb < /tmp/appdb_backup.sql
</pre>

## 5. Verification

Output of `$sudo -u postgres psql -c \l`:

<pre>
						     List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+-------------+-------------+--------+-----------+-----------------------
 appdb     | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           |                       
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           |                       
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres 
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres 
(4 rows)
</pre>

Confirmed that appdb exists again after `DROP DATABASE`  
and backing it up after recreating it again with `CREATE DATABASE appdb`  
and `appdb < /tmp/appdb_backup.sql`

## 6. Prevention
Backup/restore is now a tested, documented procedure instead of an
untested assumption.

## 7. What I'd do differently in production
I'd automate this drill on a schedule against a copy of the database, not
the live one, and alert if a scheduled restore test ever fails.
