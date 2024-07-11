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

```

Our first answer is has a misleading name, since Active Directory is a windows-based, but the tool for enumerating SMB ports (139/445) is enum4linux.

#### Enumeration
##### SMB
```

```
