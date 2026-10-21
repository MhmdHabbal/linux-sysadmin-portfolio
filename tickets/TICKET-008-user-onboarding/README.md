# TICKET-008: User Onboarding Script

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/20/2026  
**Skills practiced:** User/permission management, bash scripting

## 1. Problem Statement
New user onboarding was manual and inconsistent — missing SSH setup, wrong
permissions, and forgotten sudo group membership were all possible with a
manual checklist.

## 2. Investigation
N/A — this documents the manual process being replaced rather than a
specific incident.

## 3. Root Cause
No standardized script existed for account creation, leaving each step
dependent on a human remembering it correctly every time.

## 4. Fix
Wrote `/opt/scripts/new_user.sh`, which creates the account, sets up its
home directory and SSH key file with correct ownership and permissions,
and adds it to the `sudo` group in one command.

<pre>
    #!/bin/bash
    set -euo pipefail
    USERNAME=$1
    useradd -m -s /bin/bash "$USERNAME"
    mkdir -p /home/$USERNAME/.ssh
    echo "PUBLIC_KEY_HERE" > /home/$USERNAME/.ssh/authorized_keys
    chown -R $USERNAME:$USERNAME /home/$USERNAME/.ssh
    chmod 700 /home/$USERNAME/.ssh
    chmod 600 /home/$USERNAME/.ssh/authorized_keys
    usermod -aG sudo "$USERNAME"
    echo "User $USERNAME created."
</pre>

## 5. Verification

Output of `sudo /opt/scripts/new_user.sh testuser`:

<pre>
User testuser created.
</pre>

Output of `id testuser` after creating the `testuser`:

<pre>
uid=1002(testuser) gid=1003(testuser) groups=1003(testuser),27(sudo)
</pre>

## 6. Prevention
Onboarding is now a single repeatable command instead of a manual
checklist prone to human error.

## 7. What I'd do differently in production
In a real company I'd integrate this with Ansible or an identity system
like LDAP rather than a local script, and never hardcode a public key —
I'd pass it in as an argument instead.
