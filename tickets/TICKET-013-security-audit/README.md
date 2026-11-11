# TICKET-013: Security Audit

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/11/2026  
**Skills practiced:** Prometheus, Grafana, monitoring, VM

## 1. Problem Statement
Ran a baseline security audit to identify hardening gaps on web-01.

## 2. Investigation

The 5 warnings chose to address are:

<pre>
suggestion[]=BOOT-5122|Set a password on GRUB boot loader to prevent altering boot configuration (e.g. boot in single user mode without password)|-|-|
suggestion[]=HRDN-7230|Harden the system by installing at least one malware scanner, to perform periodic file system scans|-|Install a tool like rkhunter, chkrootkit, OSSEC, Wazuh|
suggestion[]=BOOT-5264|Consider hardening system services|Run '/usr/bin/systemd-analyze security SERVICE' for each service|-|
suggestion[]=AUTH-9286|Configure minimum password age in /etc/login.defs|-|-|
suggestion[]=AUTH-9286|Configure maximum password age in /etc/login.defs|-|-|
</pre>

## 3. Root Cause
Default Ubuntu install leaves several hardening settings at their
permissive defaults.

## 4. Fix
Addressed 5 findings from the audit one at a time (as shown in the `Investigation` section):  

First one:
<pre>
$grub-mkpasswd-pbkdf2               # To create a secure PBKDF2 hash of my password

$sudo nano /etc/grub.d/40_custom    # Adding the password to GRUB Custom Rules
				    # appending the following lines to the bottom of the file				 
				    # set superusers="MY-USERNAME"
				      password_pbkdf2 "MY-USERNAME grub.pkdf2.sha512.10000.MY_ACTUAL_HASH_STRING

$sudo nano /etc/grub.d/10_linux     # Prevent Password Prompts on Normal Boots
                                    # Adding `--unrestricted` inside the line (echo "menuentry '$(echo "$os" | grub_quote)' ${CLASS} \$menuentry_id_option ...")
				    # right before `${CLASS}`

$sudo update-grub                   # To apply changes

"Now if I want to edit GRUB while on boot I will be prompted to enter my username and password, instead of anyone being able to edit GRUB"
</pre>

Second one:

<pre>
$sudo apt install rkhunter	# To install rkhunter(rkhunter scans for rootkits, backdoors, and local exploits by comparing SHA-256 hashes of core system binaries against a known good database)
$sudo rkhunter --propupd	# Tells rkhunter my current system files are safe (baseline)
$sudo apt rkhunter --update	# Updates virus/rootkit definitions
$sudo rkhunter --check -sk	# To run a scan (The --sk flag skips the "press Enter to continue" prompts between sections)

"Now I have a malware scanner on my machine"
</pre>

Third one:

<pre>
$systemd-analyzer security		# Built-in `systemd` security analyzer to see which services are exposed.
$systemd-analyze security SERVICE_NAME	# Inspecting a specific service
$sudo systemctl edit SERVICE_NAME	# Opening the override editor for the chosen service
					# Adding the hardning rules under the `[Service]` header inside the editor
$sudo systemctl daemon-reload		# Reloading the systemd daemon
$sudo systemctl restart SERVICE_NAME	# Restarting the service
$sudo systemctl status SERVICE_NAME	# Checking if the service is active and healthy
</pre>

Fourth one:

<pre>
$sudo nano /etc/login.defs		# Opening the login configuration file
					# Changing PASS_MIN_DAYS from `0` to `1` (or above for stricter security)
$sudo chage -m 1 `username`		# Applying the rule to existing users because the new rule only affects future user accounts
$sudo chage -l `username`		# Verfying the change
</pre>

Fifth one:

<pre>
$sudo nano /etc/login.defs		# Opening the login configuration file
					# Changing PASS_MAX_DAYS from `99999` to `90` (or 180 for a less frequent rotation)
$sudo chage -M 90 `username`		# Applying the rule to existing users because the new rule only affects future user accounts
$sudo chage -l `username`		# Verifying the change
</pre>

## 5. Verification

<pre>
After running `$sudo lynis audit system` I confirmed that the five suggestions that were given before from lynis
are resolved, and my machine's score on lynis is now higher.
</pre>

## 6. Prevention
Documented this as a recurring quarterly check rather than a one-time
task.

## 7. What I'd do differently in production
I'd run Lynis as part of a CI/CD or scheduled compliance pipeline rather
than manually, and track the hardening score's trend over time instead of
checking it once.
