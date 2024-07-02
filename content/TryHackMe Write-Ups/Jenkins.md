---
title: Jenkins
draft: 
tags:
---
This room is guided, however, I will still write out my process here. Since I already know the target IP address, I will not be performing Host Discovery. I try to make it a habit with THM and HTB machines to make the target IP an environment variable with `export IP=<target IP>`
# Port Scanning

First I start with my full TCP port scan with `nmap -Pn -p- -T4 $IP`

```
nmap -Pn -p- -T4 $IP
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-02 09:08 EDT
Nmap scan report for 10.10.128.105
Host is up (0.18s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE
80/tcp   open  http
3389/tcp open  ms-wbt-server
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 171.86 seconds
```
This answers question 1.

Here we have ports 80, 3389, and 8080. These are relatively self-explanatory, but I will use my service scans anyway. I will then also move onto using gobuster for directory enumeration and looking at the webpage(s) directly.
```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-02 09:12 EDT
Nmap scan report for 10.10.128.105
Host is up (0.18s latency).

PORT     STATE SERVICE            VERSION
80/tcp   open  http               Microsoft IIS httpd 7.5
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
3389/tcp open  ssl/ms-wbt-server?
| ssl-cert: Subject: commonName=alfred
| Not valid before: 2024-07-01T13:06:59
|_Not valid after:  2024-12-31T13:06:59
|_ssl-date: 2024-07-02T13:14:34+00:00; 0s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: ALFRED
|   NetBIOS_Domain_Name: ALFRED
|   NetBIOS_Computer_Name: ALFRED
|   DNS_Domain_Name: alfred
|   DNS_Computer_Name: alfred
|   Product_Version: 6.1.7601
|_  System_Time: 2024-07-02T13:14:29+00:00
8080/tcp open  http               Jetty 9.4.z-SNAPSHOT
|_http-title: Site doesn't have a title (text/html;charset=utf-8).
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 105.23 seconds
```

Interesting, we find a robots.txt file, the hostname, and likely some service running on port 8080 called Jetty. Lets start a gobuster scan and see what we find. Meanwhile, lets check out the webpages themselves. Just going to the IP in the web browser brings up a RIP Bruce Wayne page, asking for donations - funny. I'll check `http://$IP:8080` next.

![[Pasted image 20240702082140.png]]

So, now we're at a Jenkin's login form. We can look up jenkins's default credentials, brute force this form with hydra or burp suite using names like alfred, bruce, admin, administrator, etc, or we can just handtype a few guesses first. Lets try some combinations of admin:password, admin:admin, etc.

We can also note that gobuster didn't return anything on the base IP.
```
gobuster dir -u http://$IP -w /usr/share/wordlists/amass/subdomains-top1mil-5000.txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.128.105
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/amass/subdomains-top1mil-5000.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
Progress: 5000 / 5001 (99.98%)
===============================================================
Finished
===============================================================
```

However, admin:admin worked! And we're greeted with a dashboard page.

![[Pasted image 20240702082745.png]]

So, it looks like there's a way to trigger remote code execution in the project folder via "Windows Batch Commands", but I'm not sure if this is the route. There is also a script console that will definitely run RCE on the box. 

![[Pasted image 20240702084223.png]]
