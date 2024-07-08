---
title: Relevant
draft: true
tags:
  - thm
  - oscp
  - medium
---
This room starts of suggesting to treat this as an actual pentest, stating there are multiple exploitable vulnerabilities. This machine also will not REQUIRE metasploit, which is great as I prep for OSCP which only allows metasploit on 1 machine.


# Initial Port Scan
```
map -Pn -sV -p- $IP                                     
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-08 09:47 EDT
Nmap scan report for 10.10.59.249
Host is up (0.18s latency).
Not shown: 65527 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
49663/tcp open  http          Microsoft IIS httpd 10.0
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows
```

# Enumeration
## SMB
So there's a lot to look at here. Lets start with enumerating SMB.
### Non-credentialed
```
nmap -Pn -sV --script=smb-enum-users,smb-enum-shares -p445 $IP 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-08 10:02 EDT
Nmap scan report for 10.10.59.249
Host is up (0.20s latency).

PORT    STATE SERVICE      VERSION
445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
Service Info: OS: Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.59.249\ADMIN$: 
|     Type: STYPE_DISKTREE_HIDDEN
|     Comment: Remote Admin
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.59.249\C$: 
|     Type: STYPE_DISKTREE_HIDDEN
|     Comment: Default share
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.59.249\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: Remote IPC
|     Anonymous access: <none>
|     Current user access: READ/WRITE
|   \\10.10.59.249\nt4wrksv: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Anonymous access: <none>
|_    Current user access: READ/WRITE
```

There is anonymous read/write access on the nt4wrksv drive. 

Using smbclient, we find a passwords.txt file with some encoded passwords.
```
smbclient -N \\\\10.10.59.249\\nt4wrksv
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Jul  8 10:03:27 2024
  ..                                  D        0  Mon Jul  8 10:03:27 2024
  passwords.txt                       A       98  Sat Jul 25 11:15:33 2020

                7735807 blocks of size 4096. 5137259 blocks available
smb: \> get passwords.txt 
getting file \passwords.txt of size 98 as passwords.txt (0.1 KiloBytes/sec) (average 0.1 KiloBytes/sec)
smb: \> exit

cat passwords.txt
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

We can throw those into cyberchef and select `From Base64` (just a common encoding method, we could have checked others if this came back with gibberish or didn't work)

```
Bob - !P@$$W0rD!123
Bill - Juw4nnaM4n420696969!$$$
```

We see that RDP is open so we can test these credentials for RDP access. However, we can also now use these credentials for more SMB enumeration.

### Credentialed
```
nmap -Pn -sV --script=smb-enum-users,smb-enum-shares --script-args 'smbusername=bob,smbpassword=!P@$$W0rD!123' -p445 $IP
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-08 10:33 EDT
Nmap scan report for 10.10.59.249
Host is up (0.18s latency).

PORT    STATE SERVICE      VERSION
445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
Service Info: OS: Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-enum-shares: 
|   account_used: bob
|   \\10.10.59.249\ADMIN$: 
|     Type: STYPE_DISKTREE_HIDDEN
|     Comment: Remote Admin
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.59.249\C$: 
|     Type: STYPE_DISKTREE_HIDDEN
|     Comment: Default share
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.59.249\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: Remote IPC
|     Anonymous access: <none>
|     Current user access: READ/WRITE
|   \\10.10.59.249\nt4wrksv: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Anonymous access: <none>
|_    Current user access: READ/WRITE
```

Bob has read/write access to the share that was anonymously readable. We may likely be able to upload a payload and execute it. Bill's access was not much different.

Instead of nmap scans, lets see what crackmapexec gets us for this machine on SMB.




## HTTP
Lets open our web browser and just view the webpages. There port 80 and port 49663.

Both ports just open up the default IIS landing page.