# TICKET-003: SSH Brute-Force Hardening

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/13/2026  
**Skills practiced:** SSH hardening, fail2ban, security incident response

## 1. Problem Statement
Simulated a brute-force SSH attack against web-01 to test and harden its
exposure — default config allowed unlimited password attempts on port 22.

## 2. Investigation
Confirmed default risky config: `sudo grep -E "^(PasswordAuthentication|Port|PermitRootLogin)" /etc/ssh/sshd_config`  

no output since `grep` will only show the setting name, but `sshd_config` comments out these settings, 
so we know that they are not overriden.  

If I want to show the commented out settings, I'll have to add `#?`:  
`sudo grep -E "^#?(PasswordAuthentication|Port|PermitRootLogin)" /etc/ssh/sshd_config`  

Output:

<pre>
#Port 22
#PermitRootLogin prohibit-password
#PasswordAuthentication yes
</pre>

Ran repeated failed logins from the host and confirmed they were all
accepted attempts (no lockout): `for i in {1..8}; do ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 -o BatchMode=yes wrongwronguser@<web-01-IP> "echo test"; done`  

And then verified the failed attempts on the web-01 auth.logs: `sudo grep "Failed password" /var/log/auth.log | tail -10`

<pre>
2026-10-12T20:35:06.908188+03:00 web-01 sudo: mhmd : TTY=/dev/pts/0 ; PWD=/home/mhmd ; USER=root ; COMMAND=/usr/bin/grep  Failed password /var/log/auth.log
2026-10-12T20:35:43.261827+03:00 web-01 sshd-session[4549]: Failed password for invalid user wrongwronguser from 192.168.56.1 port 58226 ssh2
2026-10-12T20:35:43.261827+03:00 web-01 sshd-session[4549]: Failed password for invalid user wrongwronguser from 192.168.56.1 port 58226 ssh2
2026-10-12T20:35:43.261827+03:00 web-01 sshd-session[4549]: Failed password for invalid user wrongwronguser from 192.168.56.1 port 58226 ssh2
2026-10-12T20:35:43.261827+03:00 web-01 sshd-session[4549]: Failed password for invalid user wrongwronguser from 192.168.56.1 port 58226 ssh2
2026-10-12T20:35:43.261827+03:00 web-01 sshd-session[4549]: Failed password for invalid user wrongwronguser from 192.168.56.1 port 58226 ssh2
2026-10-12T20:35:43.261827+03:00 web-01 sshd-session[4549]: Failed password for invalid user wrongwronguser from 192.168.56.1 port 58226 ssh2
2026-10-12T20:35:43.261827+03:00 web-01 sshd-session[4549]: Failed password for invalid user wrongwronguser from 192.168.56.1 port 58226 ssh2
2026-10-12T20:35:43.261827+03:00 web-01 sshd-session[4549]: Failed password for invalid user wrongwronguser from 192.168.56.1 port 58226 ssh2
2026-10-12T20:36:03.904384+03:00 web-01 sudo: mhmd : TTY=/dev/pts/0 ; PWD=/home/mhmd ; USER=root ; COMMAND=/usr/bin/grep Failed password /var/log/auth.log
</pre>

## 3. Root Cause
Default SSH config allowed unlimited password authentication attempts with 
no rate limiting or banning — a real internet-facing box in this state 
would eventually be compromised or DoS'd by brute force.  

Applying the actual fix surfaced several additional, genuinely real
problems beyond the baseline vulnerability itself, each with its own root
cause:  

- Two of my `sed` edits (`Port`, then separately `PasswordAuthentication`)
  silently matched zero lines on the first attempt, because the patterns
  only matched one specific starting state (e.g. already-commented) and
  the actual line in the file was in a different state. `sed` exits `0`
  and prints nothing when this happens — there is no error to notice, the
  file just silently doesn't change.  

- Port 22 kept answering even after `Port 2222` was correctly written to
  `sshd_config` and the service restarted, because `ssh.socket` — a
  separate systemd unit — was holding port 22 open independently of
  anything in `sshd_config`.  

- `systemctl restart sshd` did not appear to restart the real, running
  SSH daemon at all (the listening process's PID never changed across
  restarts); Ubuntu's actual unit is named `ssh`, not `sshd`.  

- I generated my first SSH key pair *on web-01 itself* instead of on my
  host, the machine that actually needed to prove its identity — the
  private half of a key pair must never leave the machine connecting
  FROM, and a key born on the server can't help the client authenticate.  

- After fixing that, I disabled password authentication before the key
  was actually proven working, which locked me out of both login methods
  simultaneously and required the VirtualBox console to recover.

## 4. Fix
- Generated an SSH key pair on my **host** (not the server), installed the
  public half into `~/.ssh/authorized_keys` on web-01, and confirmed
  key-based login worked — all *before* touching password authentication
  at all.  

- Used `sed -i -E` with a `^#?` pattern (rather than a literal match) for
  both `Port` and `PasswordAuthentication`, so the edit applies regardless
  of whether the line started commented, active, or set to something else.  

- Verified the change was genuinely live with `sudo sshd -T`, not just by
  reading the config file back — this is what caught the `sed` non-matches
  that otherwise would have gone unnoticed.  

- Found and disabled `ssh.socket` (`systemctl stop` + `disable`), which
  was holding port 22 open independently of `sshd_config`.  

- Restarted using the real unit name, `ssh`, not `sshd`.  

- Installed and configured fail2ban (4 attempts → 1 hour ban).  

- Moved SSH to a non-default port (2222) and disabled password
  authentication entirely (key-only).

## 5. Verification
Repeated the attack simulation after the fix and confirmed the attacking
host was banned: `$sudo fail2ban-client status sshd`

<pre>
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     6
|  `- Journal matches:  _SYSTEMD_UNIT=ssh.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned:     1
   `- Banned IP list: <IP_ADDRESS>
</pre>

Also confirmed the configuration was genuinely active, not just written to
disk: `sudo sshd -T | grep -iE "^(port|passwordauthentication)"`

<pre>
port 2222
passwordauthentication no
</pre>

## 6. Prevention
This configuration is now the baseline I apply to every new VM, including the
habit of verifying with `sshd -T` and checking for `ssh.socket` on any
new box before assuming `sshd_config` alone controls the listening port.

## 7. What I'd do differently in production
- Never trust `sed`'s exit code as proof a config change took effect —
  verify the live, effective configuration (`sshd -T` for SSH, the
  equivalent for other services) after every edit, every time.  

- Check for systemd socket activation (`systemctl status <name>.socket`)
  before assuming a service's own config file fully controls what port
  it listens on — some are overridden entirely by a separate unit.  

- On Debian/Ubuntu, use the real systemd unit names (`ssh`, not `sshd`)
  rather than assuming RHEL-style naming works identically everywhere.  

- Always generate key pairs on the client machine, never the server, and
  explicitly diff the public key on the server against the private key's
  matching public half before trusting that they correspond — two keys
  that both start `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...` can still be
  completely different keys underneath that shared boilerplate prefix.  

- Prove a replacement authentication method works end-to-end *before*
  removing the existing fallback — never the reverse. Getting this order
  wrong is what turned a straightforward hardening ticket into a
  multi-hour lockout-and-recovery exercise.  

- Remember that fail2ban (once working) can ban your OWN IP during
  repeated testing — know `fail2ban-client set <jail> unbanip <ip>` before
  you need it in a hurry.
