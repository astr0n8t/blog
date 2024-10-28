---
title: "Flag"
date: 2024-10-28T16:37:09-05:00
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
series_order: 4
---
---

This time we just get a single binary.  Running the binary says this:
```c
$ ./flag 
I will malloc() and strcpy the flag there. take it.
```

So let's open the program in pwndbg and walk through it.

Well it seems that there are no symbols defined... Weird.
```c
pwndbg> info func
All defined functions:
```

Running checksec on the binary shows that there is no PIE, so addresses will be static, but it does appear to be packed so let's try unpacking it:
```c
$ checksec flag          
    Arch:     amd64-64-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX unknown - GNU_STACK missing
    PIE:      No PIE (0x400000)
    Stack:    Executable
    RWX:      Has RWX segments
    Packer:   Packed with UPX
```

That worked:
```c
$ upx -d flag 
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2024
UPX 4.2.3       Markus Oberhumer, Laszlo Molnar & John Reiser   Mar 27th 2024

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
    887219 <-    335288   37.79%   linux/amd64   flag

Unpacked 1 file.

```

Now running `info func` gives a lot more functions:
```c
pwndbg> info func
All defined functions:

Non-debugging symbols:
0x00000000004002f8  _init
0x00000000004003d0  check_one_fd.part
0x0000000000400441  munmap_chunk.part
0x0000000000400455  group_number
0x0000000000400557  _i18n_number_rewrite
0x000000000040073c  _i18n_number_rewrite
0x0000000000400921  is_trusted_path_normalize
0x0000000000400a1a  print_search_path
0x0000000000400b69  strip
0x0000000000400bd9  group_number
0x0000000000400ca3  _i18n_number_rewrite
0x0000000000400dd0  fini
0x0000000000400de0  init_cacheinfo
0x0000000000401058  _start
0x0000000000401084  call_gmon_start
0x00000000004010a0  __do_global_dtors_aux
0x0000000000401120  frame_dummy
0x0000000000401164  main
...
```

Okay so let's set a breakpoint at main and see if we catch the strcpy call.  It looks like when we get to address `0x401195` that this is the call to strcpy and the string is there in memory which `pwndbg` helpfully shows for us so that is probably the flag:
```c
 RAX  0x6c96b0 ◂— 0x0
 RBX  0x401ae0 (__libc_csu_fini) ◂— push rbx
 RCX  0x8
 RDX  0x496628 ◂— push rbp /* '<FLAG>' */
*RDI  0x6c96b0 ◂— 0x0
 RSI  0x496628 ◂— push rbp /* '<FLAG>' */
 R8   0x1
 R9   0x3
 R10  0x22
 R11  0x0
 R12  0x401a50 (__libc_csu_init) ◂— push r14
 R13  0x0
 R14  0x0
 R15  0x0
 RBP  0x7fffffffcad0 ◂— 0x0
 RSP  0x7fffffffcac0 —▸ 0x401a50 (__libc_csu_init) ◂— push r14
*RIP  0x401195 (main+49) ◂— call 400320h
─────────────────────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────────────────────
   0x401180 <main+28>    mov    qword ptr [rbp - 8], rax
   0x401184 <main+32>    mov    rdx, qword ptr [rip + 2c0ee5h]
   0x40118b <main+39>    mov    rax, qword ptr [rbp - 8]
   0x40118f <main+43>    mov    rsi, rdx
   0x401192 <main+46>    mov    rdi, rax
 > 0x401195 <main+49>    call   400320h                       <0x400320>
```

Submitting this works.  You can also find the flag very easily after unpacking it by doing:
```sh
strings -n40 flag
```

Then the flag is the first string.  I thought it was more fun to step through the program though.




