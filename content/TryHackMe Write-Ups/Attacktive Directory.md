---
title: Attacktive Directory
draft: true
tags:
---
	# Overview

This room is not so much a CTF challenge for getting a user and root flag, but a guided tour on how to effectively hack active directory.

## Task 1 - Get Connected

This just involves connecting via VPN using Openvpn.

## Task 2 - Setup

### Installing Impacket
```
git clone https://github.com/SecureAuthCorp/impacket.git /opt/impacket
```

```
pip3 install -r /opt/impacket/requirements.txt
```

```
cd /opt/impacket/ && python3 ./setup.py install
```

#### Easy install script
```
sudo git clone https://github.com/SecureAuthCorp/impacket.git /opt/impacket
sudo pip3 install -r /opt/impacket/requirements.txt
cd /opt/impacket/ 
sudo pip3 install .
sudo python3 setup.py install
```

### Installing other useful tools
```
sudo apt install bloodhound neo4j`
```

## Task 3 - Welcome

We start off with an nmap scan here. Then there's a few questions that ask about what tools can be used to enumerate certain services. 

There's also a question of what invalid TLD, or Top Level Domain, is commonly used. Every one I've seen is `.local`

### Initial Port Scan
```
nmap -Pn -p- 10.10.12.154          
Starting Nmap 7.94SVN ( https://nmap.org )
Stats: 0:18:16 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Connect Scan Timing: About 81.97% done; ETC: 18:38 (0:04:01 remaining)
Stats: 0:21:14 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Connect Scan Timing: About 90.65% done; ETC: 18:39 (0:02:12 remaining)
Nmap scan report for 10.10.12.154
Host is up (0.18s latency).
Not shown: 65507 closed tcp ports (conn-refused)
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49675/tcp open  unknown
49679/tcp open  unknown
49685/tcp open  unknown
49697/tcp open  unknown
49827/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 1483.80 seconds
```
A domain controller has so many open ports!

### Specific Port Scan
```
nmap -Pn -sV -sC -p53,80,88,135,139,389,445,464,593,636,3268,3269,3389,5985,9389,47001,49664,49665,49666,49670,49673,49674,49675,49679,49685,49697,49827 10.10.12.154
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.12.154
Host is up (0.18s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-07-11 23:06:21Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: spookysec.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: spookysec.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: THM-AD
|   NetBIOS_Domain_Name: THM-AD
|   NetBIOS_Computer_Name: ATTACKTIVEDIREC
|   DNS_Domain_Name: spookysec.local
|   DNS_Computer_Name: AttacktiveDirectory.spookysec.local
|   Product_Version: 10.0.17763
|_
| ssl-cert: Subject: commonName=AttacktiveDirectory.spookysec.local
|_Not valid after:  2025-01-09T22:07:14
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  msrpc         Microsoft Windows RPC
49679/tcp open  msrpc         Microsoft Windows RPC
49685/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
49827/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: ATTACKTIVEDIREC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: -1s, deviation: 0s, median: -1s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 79.76 seconds
```

Our first answer is has a misleading name, since Active Directory is a windows-based, but the tool for enumerating SMB ports (139/445) is enum4linux.

#### Enumeration
##### SMB via Enum4linux
```
enum4linux -a spooky.local 
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Thu Jul 11 21:07:23 2024

 =========================================( Target Information )=========================================
                                                                                                                                                             
Target ........... spooky.local                                                                                                                              
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ============================( Enumerating Workgroup/Domain on spooky.local )============================
                                                                                                                                                             
                                                                                                                                                             
[E] Can't find workgroup/domain                                                                                                                              
                                                                                                                                                             
                                                                                                                                                             

 ================================( Nbtstat Information for spooky.local )================================
                                                                                                                                                             
Looking up status of 10.10.12.154                                                                                                                            
No reply from 10.10.12.154

 ===================================( Session Check on spooky.local )===================================
                                                                                                                                                             
                                                                                                                                                             
[+] Server spooky.local allows sessions using username '', password ''                                                                                       
                                                                                                                                                             
                                                                                                                                                             
 ================================( Getting domain SID for spooky.local )================================
                                                                                                                                                             
Domain Name: THM-AD                                                                                                                                          
Domain Sid: S-1-5-21-3591857110-2884097990-301047963

[+] Host is part of a domain (not a workgroup)                                                                                                               
                                                                                                                                                             
                                                                                                                                                             
 ===================================( OS information on spooky.local )===================================
                                                                                                                                                             
                                                                                                                                                             
[E] Can't get OS info with smbclient                                                                                                                         
                                                                                                                                                             
                                                                                                                                                             
[+] Got OS info for spooky.local from srvinfo:                                                                                                               
do_cmd: Could not initialise srvsvc. Error was NT_STATUS_ACCESS_DENIED                                                                                       


 =======================================( Users on spooky.local )=======================================
                                                                                                                                                             
                                                                                                                                                             
[E] Couldn't find users using querydispinfo: NT_STATUS_ACCESS_DENIED                                                                                         
                                                                                                                                                             
                                                                                                                                                             

[E] Couldn't find users using enumdomusers: NT_STATUS_ACCESS_DENIED                                                                                          
                                                                                                                                                             
                                                                                                                                                             
 =================================( Share Enumeration on spooky.local )=================================
                                                                                                                                                             
do_connect: Connection to spooky.local failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)                                                                      

        Sharename       Type      Comment
        ---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on spooky.local                                                                                                                 
                                                                                                                                                             
                                                                                                                                                             
 ============================( Password Policy Information for spooky.local )============================
                                                                                                                                                             
                                                                                                                                                             
[E] Unexpected error from polenum:                                                                                                                           
                                                                                                                                                             
                                                                                                                                                             

[+] Attaching to spooky.local using a NULL share

[+] Trying protocol 139/SMB...

        [!] Protocol failed: Cannot request session (Called Name:SPOOKY.LOCAL)

[+] Trying protocol 445/SMB...

        [!] Protocol failed: SAMR SessionError: code: 0xc0000022 - STATUS_ACCESS_DENIED - {Access Denied} A process has requested access to an object but has not been granted those access rights.



[E] Failed to get password policy with rpcclient                                                                                                             
                                                                                                                                                             
                                                                                                                                                             

 =======================================( Groups on spooky.local )=======================================
                                                                                                                                                             
                                                                                                                                                             
[+] Getting builtin groups:                                                                                                                                  
                                                                                                                                                             
                                                                                                                                                             
[+]  Getting builtin group memberships:                                                                                                                      
                                                                                                                                                             
                                                                                                                                                             
[+]  Getting local groups:                                                                                                                                   
                                                                                                                                                             
                                                                                                                                                             
[+]  Getting local group memberships:                                                                                                                        
                                                                                                                                                             
                                                                                                                                                             
[+]  Getting domain groups:                                                                                                                                  
                                                                                                                                                             
                                                                                                                                                             
[+]  Getting domain group memberships:                                                                                                                       
                                                                                                                                                             
                                                                                                                                                             
 ==================( Users on spooky.local via RID cycling (RIDS: 500-550,1000-1050) )==================
                                                                                                                                                             
                                                                                                                                                             
[I] Found new SID:                                                                                                                                           
S-1-5-21-3591857110-2884097990-301047963                                                                                                                     

[I] Found new SID:                                                                                                                                           
S-1-5-21-3591857110-2884097990-301047963                                                                                                                     

[+] Enumerating users using SID S-1-5-21-3532885019-1334016158-1514108833 and logon username '', password ''                                                 
                                                                                                                                                             
S-1-5-21-3532885019-1334016158-1514108833-500 ATTACKTIVEDIREC\Administrator (Local User)                                                                     
S-1-5-21-3532885019-1334016158-1514108833-501 ATTACKTIVEDIREC\Guest (Local User)
S-1-5-21-3532885019-1334016158-1514108833-503 ATTACKTIVEDIREC\DefaultAccount (Local User)
S-1-5-21-3532885019-1334016158-1514108833-504 ATTACKTIVEDIREC\WDAGUtilityAccount (Local User)
S-1-5-21-3532885019-1334016158-1514108833-513 ATTACKTIVEDIREC\None (Domain Group)

[+] Enumerating users using SID S-1-5-21-3591857110-2884097990-301047963 and logon username '', password ''                                                  
                                                                                                                                                             
S-1-5-21-3591857110-2884097990-301047963-500 THM-AD\Administrator (Local User)                                                                               
S-1-5-21-3591857110-2884097990-301047963-501 THM-AD\Guest (Local User)
S-1-5-21-3591857110-2884097990-301047963-502 THM-AD\krbtgt (Local User)
S-1-5-21-3591857110-2884097990-301047963-512 THM-AD\Domain Admins (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-513 THM-AD\Domain Users (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-514 THM-AD\Domain Guests (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-515 THM-AD\Domain Computers (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-516 THM-AD\Domain Controllers (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-517 THM-AD\Cert Publishers (Local Group)
S-1-5-21-3591857110-2884097990-301047963-518 THM-AD\Schema Admins (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-519 THM-AD\Enterprise Admins (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-520 THM-AD\Group Policy Creator Owners (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-521 THM-AD\Read-only Domain Controllers (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-522 THM-AD\Cloneable Domain Controllers (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-525 THM-AD\Protected Users (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-526 THM-AD\Key Admins (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-527 THM-AD\Enterprise Key Admins (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-1000 THM-AD\ATTACKTIVEDIREC$ (Local User)

 ===============================( Getting printer info for spooky.local )===============================
                                                                                                                                                             
do_cmd: Could not initialise spoolss. Error was NT_STATUS_ACCESS_DENIED                                                                                      


enum4linux complete
```

##### Enumerating Kerberos Users
```
kerbrute userenum -d spooky.local --dc AttacktiveDirectory.spookysec.local -t 40 lab_users.txt
```