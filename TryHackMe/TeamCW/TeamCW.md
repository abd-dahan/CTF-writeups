
# TryHackMe - Team
Platform: Tryhackme
Category: Web Enumeration / Credential exposure / Linux priviledge escalation
Difficulty: Easy-Medium

## Summary

This room / CTF simulates a small internal "Team site" that leaks credentials through a forgotten backup file, then chains that access into SSH, sudo misuse, and a writable cron script to escalate to root. It's a good example of how a handful of small, individually "minor" misconfigurations (that being the forgotten .old file) can have a large impact in the end, and that being root access to admin. every step in this chain would have left a distinct trail in logs, which I've noted throughout.

## Tools used
dirsearch — initial directory enumeration
ffuf — targeted fuzzing on the /scripts/ path
ftp — retrieving files from an exposed FTP share
Browser LFI (script.php?page=) — reading local files off the dev vhost
ssh — authenticating with a recovered private key
python3 pty.spawn — shell stabilization
nc (netcat) — catching a reverse shell
nano — editing the writable cron script

## Objective 
Enumerate a web application, recover leaked credentials, pivot across two vhosts, and escalate from an unprivileged user to root.

## Set-up

if you're using the TryHackMe attack box, you can skip this section. Otherwise, to actually interact with the machine you're going to need to install openVPN from https://tryhackme.com/access and follow the steps to enable the VPN, secondly, you'll need to get the IP of the Lab machine and put it in your /etc/hosts file like the following:

![Setting up /etc/hosts](./screenshots/1.png)
![Setting up /etc/hosts](./screenshots/2.png)
