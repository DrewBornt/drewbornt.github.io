---
title: Incident Response Recovery
draft: true
tags:
---
# About
The NIST Incident Response Steps detail in order Preparation, Identification, Containment, Eradication, Recovery, and Lessons Learned. This is a proper order for organizations that are capable of having their own SOC (Security Operations Center) department or have third-partied for services of one. However, not all corporations and organizations have such a refined cybersecurity implementation. Because of this, on the IR's I've been a part of, my team and I are coming in after large scale encryption events have taken down an organization. For this reason, my notes and processes written will be from solely the Recovery portion of NIST's IR Steps as a separate entity from the affected company.

Before this, I only have had loosely cobbled together notes. As such, as you read through this, understand this is a living "document", subject to change as I fix things and/or gain more experience. I will also not be going over anything my employer deems "proprietary". 

## The Steps
1. [[Information Gathering]]
2. [[Network Isolation]]
3. [[Forensics Collection]]
4. [[Infrastructure Rebuild and Restore]] (Potentially Skippable)
5. [[Domain Cleanup]]
6. [[Recovery, Rebuild, and Decrypt]]
7. [[Sanitization]]

#### Task Priorities 
> [!note]
> The following is a rough draft of task priorities. Each tier requires the tier before it to either be finished or started. You can't work on a VM of a domain controller without a working hypervisor, can you? I may come up with better terminology later.

##### Tier 0
Information Gathering, Forensics Collection
##### Tier 1
Network Isolation, Infrastructure Rebuild and Restore
##### Tier 2
Domain Cleanup, [[DNS Cleanup]], Backup Restores, Decryption, EDR/XDR,etc Monitoring
##### Tier 3
Sanitization, Reimaging, Hardening
##### Tier 4
Move to Production
##### Tier 5
Fixing Broken Servers, Helpdesk-esque asks



