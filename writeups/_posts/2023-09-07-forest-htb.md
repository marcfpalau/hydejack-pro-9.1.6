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

Forest in an easy difficulty Windows Domain Controller (DC), for a domain in which Exchange Server has been installed. 
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
![800x400](/assets/img/blog/nmapforest.png "Large example image")
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

TODO...
