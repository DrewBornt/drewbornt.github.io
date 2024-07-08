---
title: Information Gathering
draft: true
tags:
---
# About
The start of an IR tends to be a scoping call done between an insurance company (the company I work for is partnered with), the client, and a few department leads. This is the first source of information gotten from the client about what way they were attacked. Is there file-level encryption? Hypervisor-level encryption? What is the extent of the damage and what has the client done already?

Does the client have backups of the affected machines? Did their backups get wiped out? I've seen many instances where clients had their VEEAM server authenticating with domain credentials and their backups were cleared out. Same with their vCenter hosts that have been "domain-joined".

There needs to be a ground zero to build up from. Discovering if there is a domain controller that is still functional is crucial for domain environments.