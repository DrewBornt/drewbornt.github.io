---
title: Kioptrix Level 1
draft: false
tags:
  - easy
  - vulnhub
  - oscp
  - pnpt
---
# Overview
I was planning on testing against this machine as part of my OSCP practice. However, I have decided to jump at attempting the PNPT before the OSCP. In the course material, this is one of the boxes he goes over exploiting as practice and prep for the PNPT exam to nail down our methodology of scanning, enumeration,  searching for exploits, and tailoring them. It's a bunch of enumeration with what ultimately leads up to an easy metasploit exploitation, since the deprecated C code needs too much fixing even with the ~4-5 year old fixes. 

Also - no proof on this box to show it was exploited! Rude.
# Nmap Scan #1 
```
nmap -Pn -p- -T5 $IP
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.42.1.28
Host is up (0.00015s latency).
Not shown: 65529 closed tcp ports (conn-refused)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
139/tcp   open  netbios-ssn
443/tcp   open  https
32768/tcp open  filenet-tms

Nmap done: 1 IP address (1 host up) scanned in 2.79 seconds
```

# Nmap Scan #2
```
sudo nmap -Pn -p22,80,111,139,443,32768 -sV -sC -O $IP
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-12 12:13 EDT
Nmap scan report for 10.42.1.28
Host is up (0.00010s latency).

PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 2.9p2 (protocol 1.99)
| ssh-hostkey: 
|   1024 b8:74:6c:db:fd:8b:e6:66:e9:2a:2b:df:5e:6f:64:86 (RSA1)
|   1024 8f:8e:5b:81:ed:21:ab:c1:80:e1:57:a3:3c:85:c4:71 (DSA)
|_  1024 ed:4e:a9:4a:06:14:ff:15:14:ce:da:3a:80:db:e2:81 (RSA)
|_sshv1: Server supports SSHv1
80/tcp    open  http        Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-title: Test Page for the Apache Web Server on Red Hat Linux
| http-methods: 
|_  Potentially risky methods: TRACE
111/tcp   open  rpcbind     2 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2            111/tcp   rpcbind
|   100000  2            111/udp   rpcbind
|   100024  1          32768/tcp   status
|_  100024  1          32769/udp   status
139/tcp   open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp   open  ssl/https   Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC4_128_WITH_MD5
|     SSL2_DES_64_CBC_WITH_MD5
|     SSL2_RC4_64_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|_    SSL2_RC2_128_CBC_WITH_MD5
|_http-title: 400 Bad Request
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2009-09-26T09:32:06
|_Not valid after:  2010-09-26T09:32:06
32768/tcp open  status      1 (RPC #100024)
MAC Address: 08:00:27:71:27:F0 (Oracle VirtualBox virtual NIC)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 2.4.X
OS CPE: cpe:/o:linux:linux_kernel:2.4
OS details: Linux 2.4.9 - 2.4.18 (likely embedded)
Network Distance: 1 hop

Host script results:
|_nbstat: NetBIOS name: KIOPTRIX, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_clock-skew: 3h59m56s
|_smb2-time: Protocol negotiation failed (SMB2)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.03 seconds
```

# Enumeration

## HTTP
### nikto
```
nikto -h http://$IP                              
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.42.1.28
+ Target Hostname:    10.42.1.28
+ Target Port:        80
---------------------------------------------------------------------------
+ Server: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
+ /: Server may leak inodes via ETags, header found with file /, inode: 34821, size: 2890, mtime: Wed Sep  5 23:12:46 2001. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-1418
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ /: Apache is vulnerable to XSS via the Expect header. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2006-3918
+ mod_ssl/2.8.4 appears to be outdated (current is at least 2.9.6) (may depend on server version).
+ Apache/1.3.20 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ OpenSSL/0.9.6b appears to be outdated (current is at least 3.0.7). OpenSSL 1.1.1s is current for the 1.x branch and will be supported until Nov 11 2023.
+ OPTIONS: Allowed HTTP Methods: GET, HEAD, OPTIONS, TRACE .
+ /: HTTP TRACE method is active which suggests the host is vulnerable to XST. See: https://owasp.org/www-community/attacks/Cross_Site_Tracing
+ Apache/1.3.20 - Apache 1.x up 1.2.34 are vulnerable to a remote DoS and possible code execution.
+ Apache/1.3.20 - Apache 1.3 below 1.3.27 are vulnerable to a local buffer overflow which allows attackers to kill any process on the system.
+ Apache/1.3.20 - Apache 1.3 below 1.3.29 are vulnerable to overflows in mod_rewrite and mod_cgi.
+ mod_ssl/2.8.4 - mod_ssl 2.8.7 and lower are vulnerable to a remote buffer overflow which may allow a remote shell.
+ /usage/: Webalizer may be installed. Versions lower than 2.01-09 vulnerable to Cross Site Scripting (XSS). See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2001-0835
+ /manual/: Directory indexing found.
+ /manual/: Web server manual found.
+ /icons/: Directory indexing found.
+ /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/
+ /test.php: This might be interesting.
+ /#wp-config.php#: #wp-config.php# file found. This file contains the credentials.
+ 8908 requests: 0 error(s) and 20 item(s) reported on remote host
---------------------------------------------------------------------------
+ 1 host(s) tested
```

#### Interesting Nikto Findings

`mod_ssl/2.8.4 - mod_ssl 2.8.7 and lower are vulnerable to a remote buffer overflow which may allow a remote shell.`

### dirbuster
```
DirBuster 1.0-RC1 - Report
http://www.owasp.org/index.php/Category:OWASP_DirBuster_Project
--------------------------------

http://$IP
--------------------------------
Directories found during testing:

Dirs found with a 403 response:

/cgi-bin/
/doc/

Dirs found with a 200 response:

/
/icons/
/manual/
/manual/mod/
/icons/small/
/usage/
/mrtg/
/manual/mod/mod_perl/
/manual/mod/mod_ssl/


--------------------------------
Files found during testing:

Files found with a 200 responce:

/test.php
/usage/usage_200909.html
/mrtg/mrtg.html
/mrtg/unix-guide.html
/mrtg/nt-guide.html
/mrtg/cfgmaker.html
/mrtg/indexmaker.html
/mrtg/reference.html
/mrtg/faq.html
/mrtg/forum.html
/mrtg/contrib.html
/mrtg/mrtg-rrd.html
/mrtg/logfile.html
/mrtg/mibhelp.html
/mrtg/squid.html
/mrtg/webserver.html
/manual/mod/mod_ssl/ssl_overview.html
/manual/mod/mod_ssl/index.html
/manual/mod/mod_ssl/ssl_intro.html
/manual/mod/mod_ssl/ssl_reference.html
/manual/mod/mod_ssl/ssl_compat.html
/manual/mod/mod_ssl/ssl_howto.html
/manual/mod/mod_ssl/ssl_faq.html
/manual/mod/mod_ssl/ssl_glossary.html


--------------------------------

```

### burp suite
- http headers disclose server version

### Wappalyzer
![[KioptrixLevel1Wappalyzer.png]]

## SMB / Samba
### metasploit
![[KioptrixMetasploitSMBScan.png]]

# Exploitation
## Nessus Scan
![[KioptrixNessusScan.png]]

## Searchsploit Findings
### SMB/Samba
```
searchsploit samba 2.2                
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Samba 2.0.x/2.2 - Arbitrary File Creation                                                                                  | unix/remote/20968.txt
Samba 2.2.0 < 2.2.8 (OSX) - trans2open Overflow (Metasploit)                                                               | osx/remote/9924.rb
Samba 2.2.2 < 2.2.6 - 'nttrans' Remote Buffer Overflow (Metasploit) (1)                                                    | linux/remote/16321.rb
Samba 2.2.8 (BSD x86) - 'trans2open' Remote Overflow (Metasploit)                                                          | bsd_x86/remote/16880.rb
Samba 2.2.8 (Linux Kernel 2.6 / Debian / Mandrake) - Share Privilege Escalation                                            | linux/local/23674.txt
Samba 2.2.8 (Linux x86) - 'trans2open' Remote Overflow (Metasploit)                                                        | linux_x86/remote/16861.rb
Samba 2.2.8 (OSX/PPC) - 'trans2open' Remote Overflow (Metasploit)                                                          | osx_ppc/remote/16876.rb
Samba 2.2.8 (Solaris SPARC) - 'trans2open' Remote Overflow (Metasploit)                                                    | solaris_sparc/remote/16330.rb
Samba 2.2.8 - Brute Force Method Remote Command Execution                                                                  | linux/remote/55.c
Samba 2.2.x - 'call_trans2open' Remote Buffer Overflow (1)                                                                 | unix/remote/22468.c
Samba 2.2.x - 'call_trans2open' Remote Buffer Overflow (2)                                                                 | unix/remote/22469.c
Samba 2.2.x - 'call_trans2open' Remote Buffer Overflow (3)                                                                 | unix/remote/22470.c
Samba 2.2.x - 'call_trans2open' Remote Buffer Overflow (4)                                                                 | unix/remote/22471.txt
Samba 2.2.x - 'nttrans' Remote Overflow (Metasploit)                                                                       | linux/remote/9936.rb
Samba 2.2.x - CIFS/9000 Server A.01.x Packet Assembling Buffer Overflow                                                    | unix/remote/22356.c
Samba 2.2.x - Remote Buffer Overflow                                                                                       | linux/remote/7.pl
Samba < 2.2.8 (Linux/BSD) - Remote Code Execution                                                                          | multiple/remote/10.c
Samba < 3.0.20 - Remote Heap Overflow                                                                                      | linux/remote/7701.txt
Samba < 3.6.2 (x86) - Denial of Service (PoC)                                                                              | linux_x86/dos/36741.py
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

### mod_ssl
```
searchsploit mod_ssl
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Apache mod_ssl 2.0.x - Remote Denial of Service                                                                            | linux/dos/24590.txt
Apache mod_ssl 2.8.x - Off-by-One HTAccess Buffer Overflow                                                                 | multiple/dos/21575.txt
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuck.c' Remote Buffer Overflow                                                       | unix/remote/21671.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (1)                                                 | unix/remote/764.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (2)                                                 | unix/remote/47080.c
Apache mod_ssl OpenSSL < 0.9.6d / < 0.9.7-beta2 - 'openssl-too-open.c' SSL2 KEY_ARG Overflow                               | unix/remote/40347.txt
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

## Execution

Metasploit can easily be used to exploit this machine with the samba 2.2 remote overflow module. Simply search "samba 2.2", "use" the linux one, set the RHOSTS, then set the payload to `linux/x86/shell/reverse_tcp` and exploit it.

However, there's manual ways to root this box as well. Unfortunately, the c code for this box - and the box itself - are majorly outdated. I'm not going to spend the time down the rabbit hold of fixing all of this c code because it is not compatible with OpenSSL 3.0 - even with Hackersploits's "fix" (which was a fix ~4-5 years ago).