---
title: "pwnable.kr walkthrough 01: fd"
date: 2024-10-21T16:02:11-05:00
draft: false
author: "Nathan Higley"
authorLink: ""
description: ""

tags: [linux, pwn, ctf]
categories: [pwn, ctf, linux]

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

Welcome to the first installation in my walkthrough of the https://pwnable.kr Toddler's Bottle category of challenges.  The idea is that as I start to go through all of these challenges, I'm going to make a walkthrough and post it here.  Hopefully I will be able to keep this up once a week and in this way finish out this series as well as keep myself motivated to finish the first category.

Let's begin.

# fd

This challenge starts with an SSH login provided.

On login we are presented with:
```
 ____  __    __  ____    ____  ____   _        ___      __  _  ____
|    \|  |__|  ||    \  /    ||    \ | |      /  _]    |  |/ ]|    \
|  o  )  |  |  ||  _  ||  o  ||  o  )| |     /  [_     |  ' / |  D  )
|   _/|  |  |  ||  |  ||     ||     || |___ |    _]    |    \ |    /
|  |  |  `  '  ||  |  ||  _  ||  O  ||     ||   [_  __ |     \|    \
|  |   \      / |  |  ||  |  ||     ||     ||     ||  ||  .  ||  .  \
|__|    \_/\_/  |__|__||__|__||_____||_____||_____||__||__|\_||__|\_|

- Site admin : daehee87@khu.ac.kr
- irc.netgarage.org:6667 / #pwnable.kr
- Simply type "irssi" command to join IRC now
- files under /tmp can be erased anytime. make your directory under /tmp
- to use peda, issue `source /usr/share/peda/peda.py` in gdb terminal
You have mail.
Last login: Mon Oct 21 15:44:43 2024 from 84.64.199.146
fd@pwnable:~$ ls
fd  fd.c  flag
fd@pwnable:~$ ls -al
total 40
drwxr-x---   5 root   fd   4096 Aug 31 16:09 .
drwxr-xr-x 116 root   root 4096 Oct 30  2023 ..
d---------   2 root   root 4096 Jun 12  2014 .bash_history
-r-sr-x---   1 fd_pwn fd   7322 Jun 11  2014 fd
-rw-r--r--   1 root   root  418 Jun 11  2014 fd.c
-r--r-----   1 fd_pwn root   50 Jun 11  2014 flag
-rw-------   1 root   root  128 Oct 26  2016 .gdb_history
dr-xr-xr-x   2 root   root 4096 Dec 19  2016 .irssi
drwxr-xr-x   2 root   root 4096 Oct 23  2016 .pwntools-cache
```

Since fd is a setuid binary owned by fd_pwn and the flag is also owned by that user, we should be able to read the flag if we can exploit that binary somehow.

If we look at fd.c we get:
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
        if(argc<2){
                printf("pass argv[1] a number\n");
                return 0;
        }
        int fd = atoi( argv[1] ) - 0x1234;
        int len = 0;
        len = read(fd, buf, 32);
        if(!strcmp("LETMEWIN\n", buf)){
                printf("good job :)\n");
                system("/bin/cat flag");
                exit(0);
        }
        printf("learn about Linux file IO\n");
        return 0;

}
```

This program takes the first argument given to it, then performs the `atoi()` function on it.

Reading the linux man page on `atoi()` `man 3 atoi` we read:
```
atoi, atol, atoll - convert a string to an integer
```

So this expects an integer that when subtracted with `0x1234` will be a file descriptor.

Let's try using `0x1234` directly or `4660` in decimal since that will be file descriptor 0 AKA stdin (0 is stdin, 1 is stdout, and 2 is stderr).

When we use that and paste in the phrase "LETMEWIN" followed by enter we get the flag:
```
fd@pwnable:~$ ./fd 4660
LETMEWIN
good job :)
<FLAG>
```

Quick and simple, and if you want to learn more about Linux file descriptors, here's a page from a Harvard Computer Science class going over them https://cs61.seas.harvard.edu/site/ref/file-descriptors/#gsc.tab=0
