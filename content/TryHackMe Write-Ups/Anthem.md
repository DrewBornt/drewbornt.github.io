---
title: Anthem
draft: 
tags:
  - thm
  - oscp
  - easy
---
Starting anthem is exciting. This is one of the last few *easy* Windows boxes listed for OSCP prep on the list I found on Reddit. I've already passed eJPT so Expose, Alfred, Kenobi, and Steel Mountain have all been something I'm comfortable with doing without relying on others' write-ups. My biggest weakness is not know all of the tools out there available at my disposal. I am going to continue with my current methodology. 

# Port Scanning
 As always, start with a full TCP port scan on the box with `nmap -p- -Pn $IP`
 While this scans, we can already assume some of the port numbers for the questions like 80 and 3389.
## Full TCP Port Scan
 ```
 nmap -Pn -p- -T4 $IP                                                  
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-02 13:49 EDT
Nmap scan report for 10.10.61.52
Host is up (0.18s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT     STATE SERVICE
80/tcp   open  http
3389/tcp open  ms-wbt-server

Nmap done: 1 IP address (1 host up) scanned in 205.38 seconds
```
Funny enough, that's all that's open.

## Service Scan
```
nmap -Pn -p80,3389 -sC -sV $IP
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-02 13:56 EDT
Nmap scan report for 10.10.61.52
Host is up (0.17s latency).

PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2024-07-02T17:57:42+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=WIN-LU09299160F
| Not valid before: 2024-07-01T17:44:15
|_Not valid after:  2024-12-31T17:44:15
| rdp-ntlm-info: 
|   Target_Name: WIN-LU09299160F
|   NetBIOS_Domain_Name: WIN-LU09299160F
|   NetBIOS_Computer_Name: WIN-LU09299160F
|   DNS_Domain_Name: WIN-LU09299160F
|   DNS_Computer_Name: WIN-LU09299160F
|   Product_Version: 10.0.17763
|_  System_Time: 2024-07-02T17:56:40+00:00
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 79.94 seconds
```

Okay, nothing overwhelmingly crazy here. Since we have a web service, lets check it out in the browser, but not after starting a gobuster scan.
`gobuster dir -u http://$IP -w /usr/share/wordlists/amass/subdomains-top1mil-5000.txt`

We find the website name at the bottom of the webpage. We can add this to `/etc/hosts` and possibly do subdomain enumeration later.
![[AnthemDomainName.png]]
This ends up being the answer to one of the room's questions.

We can also just quickly check if the site has a robot.txt, which it does!

We answer two questions with this, as one is the potential password and the other is the CMS in use.

About this time, the gobuster scan is done.

# Enumeration
## Gobuster
```
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.61.52
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/amass/subdomains-top1mil-5000.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/blog                 (Status: 200) [Size: 5389]
/search               (Status: 200) [Size: 3464]
/archive              (Status: 301) [Size: 123] [--> /blog/]
/rss                  (Status: 200) [Size: 1869]
Progress: 1431 / 5001 (28.61%)[ERROR] Get "http://10.10.61.52/host2123": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.61.52/libra": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.61.52/rose": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.61.52/cloud1": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.61.52/album": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
Progress: 1436 / 5001 (28.71%)[ERROR] Get "http://10.10.61.52/3": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.61.52/antares": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.61.52/www.a": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
Progress: 1439 / 5001 (28.77%)[ERROR] Get "http://10.10.61.52/ipv6": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.61.52/bridge": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
/sitemap              (Status: 200) [Size: 1030]
/install              (Status: 302) [Size: 126] [--> /umbraco/]
Progress: 2754 / 5001 (55.07%)[ERROR] Get "http://10.10.61.52/m.": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
Progress: 2852 / 5001 (57.03%)[ERROR] Get "http://10.10.61.52/ns2.cl.bellsouth.net.": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
Progress: 2940 / 5001 (58.79%)[ERROR] Get "http://10.10.61.52/ns1.viviotech.net.": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
Progress: 3103 / 5001 (62.05%)[ERROR] Get "http://10.10.61.52/ns3.cl.bellsouth.net.": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
/tags                 (Status: 200) [Size: 3539]
Progress: 5000 / 5001 (99.98%)
===============================================================
Finished
===============================================================
```

Sitemap will lead us to a name in archive/a-cheers-to-our-it-department/
- James Orchard Halliwell
There's also 
- Jane Doe
On one of the blog posts we find Jane Doe's email as `JD@anthem.com`
Does this make James Halliwell the site administrator? is his email `JOH@anthem.com` or just `JH@anthem.com`?

Going to Jane Doe's page finds us a flag. Apparently it's flag number 3. Looking at the hints, we also are asked if we've inspected the source code yet.

After inspecting the main page's source code, using CTRL+F and typing in `TMH`, we find a flag that ends up being flag #2. But the hint was for #1? Interesting.

After looking through a bunch of different page sources, we finally find flag #1 in `/archive/we-are-hiring/`
And flag #4 finds itself in `/archive/a-cheers-to-our-it-department/`

A little ridiculous. Still haven't found a set of credentials yet. Wait...

The hint is to use google. And the blog post says they've wrote a poem about them.

Google searching the poem finds us the administrator's name. 

And his initials make up his email, if Jane Doe is anything to go off of.
# Initial Foothold
Going to the login page found with `http://$IP/umbraco`, we can now attempt to log in with an admin's credentials.

The email and the hinted to password in the robots.txt file doesn't work here.

RDP was open on the box, so lets attempt using the credentials to access the box via RDP.
(My anthem box expired from stepping away for a bit, so had to relaunch it.)
`xfreerdp /u:sg /p:<password found previously> /v:10.10.143.72`

And we're in with another flag on the desktop! 

Our hint to the last flag is that it is hidden.

Lets toggle hidden files in file explorer and then look for hidden files or folders.

Turns out there's a hidden "backup" folder on the C drive. In it, a restore.txt file.

We have to change the security permissions on it to give ourselves read permissions, but in it we find what looks like a password. 

Lets try using RDP again with these new credentials and the builtin Administrator account.

# Privilege Escalation

We RDP in with the found credentials, and are greeted with the desktop of the Administrator account and the root flag!

![[AnthemComplete.png]]