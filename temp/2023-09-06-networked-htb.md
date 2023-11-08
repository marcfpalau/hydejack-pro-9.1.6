---
layout: post
title: HackTheBox Networked Walkthrough
image: /assets/img/blog/networked/networked.png
accent_image: 
  background: url('/assets/img/blog/networked/networked.png') center/cover
  overlay: false
accent_color: '#ccc'
theme_color: '#ccc'
description: >
  In this post you will find a step by step resolution walkthrough of the Networked machine on HTB platform 2023.
invert_sidebar: true
---

# HackTheBox Networked Walkthrough
Networked is an Easy difficulty Linux box vulnerable to file upload bypass, leading to code execution. Due to improper sanitization, 
a crontab running as the user can be exploited to achieve command execution. The user has privileges to execute a network configuration 
script, which can be leveraged to execute commands as root. 
* toc
{:toc}

## Enumeration
To enumerate the machine a first nmap scan to discover the open TCP ports on the machine was performed. Also when the open ports 
were retrieved a more in depth scan including fingerprinting was issued. With the following commands:
~~~bash
# First nmap scan
sudo nmap -sS -p- --min-rate 500 --open 10.10.10.56 -Pn -n -oG recon
# Exhaustive scan of the open ports
nmap -p80,2222 -sCV -Pn -n -oN openports 10.10.10.56
~~~

The results obtained by this scans showed some interesting open ports:
![800x400](/assets/img/blog/shocker/nmap.png "Nmap enumeration")
* Port 80 is open and is hosting a HTTP server, Apache httpd 2.4.18 
* Port 2222 is open and from the obtained banner looks like a ssh service version: OpenSSH 7.2p2

From the nmap scan we can see this is a linux machine, probably a Ubuntu distro.
## Web enumeration
As there are only two ports open and ssh is often not vulnerable by itself, we started enmumeration via TCP 80 port. First fingerprinting the website returns similar information as the obtained executing the nmap scripts. 
Moving on and visiting the website we obtain... ????coronavirus???? Anyway...

Further enunmeration of the site via directory bruteforcing for hidden directories and files with gobuster using a +220k entries wordlist and appending extensions as sh, php & txt was performed.


## Getting a foothold

Great we are inside! 😈

At this point we got the flag located at `C:\Users\svc-alfresco\Desktop\user.txt`
## Post-Exploitation enumeration


## Privilege Escalation

At this point we successfully pwned the machine and we have got complete control of the system 💀💀💀

Last but not least we retrieved the flag located at `C:\Users\Administrator\Desktop\root.txt`
