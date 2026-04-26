---
title: "pwnable.kr walkthrough 02: collision"
date: 2024-10-22T21:07:19-05:00
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

series: ["pwnable.kr"]
series_order: 2
---

# Collision

When we login we are prsented with:
```
col@pwnable:~$ ls -al
total 36
drwxr-x---   5 root    col     4096 Oct 23  2016 .
drwxr-xr-x 116 root    root    4096 Oct 30  2023 ..
d---------   2 root    root    4096 Jun 12  2014 .bash_history
-r-sr-x---   1 col_pwn col     7341 Jun 11  2014 col
-rw-r--r--   1 root    root     555 Jun 12  2014 col.c
-r--r-----   1 col_pwn col_pwn   52 Jun 11  2014 flag
dr-xr-xr-x   2 root    root    4096 Aug 20  2014 .irssi
drwxr-xr-x   2 root    root    4096 Oct 23  2016 .pwntools-cache
```

This is the typical scenario where the binary is setuid to the user that owns the flag.  Exploit the binary, view the flag.

Taking a look at col.c we see:
```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
        int* ip = (int*)p;
        int i;
        int res=0;
        for(i=0; i<5; i++){
                res += ip[i];
        }
        return res;
}

int main(int argc, char* argv[]){
        if(argc<2){
                printf("usage : %s [passcode]\n", argv[0]);
                return 0;
        }
        if(strlen(argv[1]) != 20){
                printf("passcode length should be 20 bytes\n");
                return 0;
        }

        if(hashcode == check_password( argv[1] )){
                system("/bin/cat flag");
                return 0;
        }
        else
                printf("wrong passcode.\n");
        return 0;
}
```

## Looking at it in GDB

Using gdb we can run through this in a more friendly environment

```
col@pwnable:~$ gdb
(gdb) source /usr/share/peda/peda.py
gdb-peda$ file col
gdb-peda$ r AAAAAAAAAAAAAAAAAAAA
```

And right before it compares the output of the check_password function we see:
```c
   0x804855c <main+143>:        mov    DWORD PTR [esp],eax
   0x804855f <main+146>:        call   0x8048494 <check_password>
   0x8048564 <main+151>:        mov    edx,DWORD PTR ds:0x804a020
=> 0x804856a <main+157>:        cmp    eax,edx
   0x804856c <main+159>:        jne    0x8048581 <main+180>
   0x804856e <main+161>:        mov    DWORD PTR [esp],0x80486bb
   0x8048575 <main+168>:        call   0x80483b0 <system@plt>
   0x804857a <main+173>:        mov    eax,0x0
                  
```

The values in edx and eax need to be the same:
```
EAX: 0x46464645
EDX: 0x21dd09ec
```

So it appears that maybe we could simply pass in a very similar value to the hashcode in the source code?
```
0x41414141 -> 0x46464645
0x42424242 -> 0x4b4b4b4a
0x???????? -> 0x21dd09ec
```

## Reverse Engineering the Code

Looking closer at the hash function, what is actually happening is that the char pointer is being cast as an int pointer.  So essentially our 20 byte string is really being interpreted as a five item integer array.
```
AAAAAAAAAAAAAAAAAAAA 
-> 
[0x41414141, 0x41414141, 0x41414141, 0x41414141, 0x41414141]
```
Which should explain the behavior we saw:
```
0x41414141+0x41414141+0x41414141+0x41414141+0x41414141=0x146464645
```
And then since the computer will only look at the first four bytes it becomes
```
0x146464645 -> 0x46464645
```

## Solution?

So what we need is five numbers, when added together, that will produce `0x21dd09ec`.

A simple way to do this is subtract `0x01010101` four times and then take the remainder as the last bytes
```
0x21dd09ec - 0x01010101*4 = 0x1DD905E8
```

The last trick to remember here is that x86 computers work in Little Endian format, so the bytes are going to be swapped around.  Instead of putting in the string representation of `0x1DD905E8` we need the string representation of `0xE805D91D` for the computer to place it in memory in order.

We can use echo to take this all in and send it to the program:
```
echo -en '\xe8\x05\xd9\x1d\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01'
```

Putting it all together gives us the flag:
```
./col $(echo -en '\xe8\x05\xd9\x1d\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01')
<FLAG>
```

This challenge was a good introduction to the fact that memory can be represented many different ways.  If you want to learn more about different C data types, I recommend looking at [this wikipedia article](https://en.wikipedia.org/wiki/C_data_types).
