---
title: "Common Sense Writeup"
subtitle: "SummitCTF 2023"
date: 2023-04-17T18:45:00-05:00
lastmod: 2023-04-17T18:45:00-05:00
draft: false
author: "Nathan Higley"
authorLink: ""
description: ""

tags: [ctf, writeup]
categories: [Writeups, SummitCTF 2023]

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

Common Sense was the only reverse engineering challenge in Virginia Tech's online 2023 SummitCTF competition.  I spent roughly five of the eight hours of the competition working on this challenge and was only finally able to solve it five minutes after the competition ended.  

The challenge simply provided two files: **a.out** and **flag.txt**.

## flag.txt

As you can see below, flag.txt is a string of characters which don't make a lot of sense on their own.

```
;]|^_j8d9YOZ;mrI]F=|:P5^iP7IPIKF4\4IPI5AUb99
 
```
I initially suspected some type of xor operation happening here, but I would not be able to confirm that for a while.

## a.out

Looking at the file a.out, it appeared to be an ELF binary, and running it simply resulted in a segmentation fault.  Opening it up in Ghidra revealed that it was in fact an ELF binary with debugging symbols stripped in order to increase the difficulty to reverse it.

### Finding main

The first thing to do with such a binary is locate the syscall to libc_start_main which will inform us as to the location of the main function.

Using the Ghidra decompiler and walking through the various functions, eventually we reach the following:
```
void FUN_001010d0(undefined8 param_1,undefined8 param_2,undefined8 param_3)
{
  undefined8 unaff_retaddr;
  undefined auStack_8 [8];
  
  __libc_start_main(main,unaff_retaddr,&stack0x00000008,0,0,param_3,auStack_8);
  do {
                    /* WARNING: Do nothing block with infinite loop */
  } while( true );
}
```

In this case, I renamed the main function to simply "main", but you can see that in the function parameters to libc_start_main, that the first argument will be the name of the main function.

### Reversing main

#### main.c

When I initially looked at the decompiled main function, it was very confusing.  Ghidra had wrongly guessed about quite a few different data types and all of the functions and variables lacked helpful names.  I spent some time walking through the function and cleaning it up to where it made sense, and I could begin to understand it.

As you can see below, the purpose of main is to check a string against quite a few different things.
```
int main(int argc,char **argv)

{
  int return_code;
  char flag_buffer [52];
  int flag_equal;
  FILE *flag_fd;
  char third_and_fifth;
  int odd_xor;
  int check_high_bits;
  char greater_than_7;
  
  greater_than_7 = check_if_arg1_greater_than_7(argc,argv);
  check_high_bits = check_size_arg1(argv[1]);
  if ((greater_than_7 == 1) && (check_high_bits != 0)) {
    odd_xor = check_odd_bitwise_and(argv[(long)argc + -1]);
    third_and_fifth = add_char_4_and_2(argv[1]);
    if ((odd_xor == 0) && (third_and_fifth == 'm')) {
      if (((argv[1][6] < '\"') || ('.' < argv[1][6])) || (argv[1][7] != '0')) {
        return_code = 1;
      }
      else {
        flag_fd = fopen("flag.txt","r");
        fgets(flag_buffer,0x2e,flag_fd);
        flag_equal = compare(flag_buffer,argv[1],1);
        if (flag_equal == 0) {
          decode_msg();
          return_code = 0;
        }
        else {
          return_code = 1;
        }
      }
    }
    else {
      return_code = 1;
    }
  }
  else {
    return_code = 1;
  }
  return return_code;
}
```

#### check_if_arg1_greater_than_7.c

The first function call which you can see below, ensures that there is exactly one argument supplied to the function and that it is a string of more than seven characters.
```
size_t check_if_arg1_greater_than_7(int argc,char **argv)
{
  size_t len_arg1;
  
  if (argc == 2) {
    len_arg1 = strlen(argv[1]);
    if (7 < len_arg1) {
      len_arg1 = 1;
    }
  }
  else {
    len_arg1 = 0;
  }
  return len_arg1;
}
```

#### check_size_arg1.c

The second function call checks if length of the string argument is an integer with all bits from the third and upward are set.  The easiest length to use to satisfy this requirement is eight characters since that will be "1000" in binary which will simply be "1" when bit shifted by three.
```
uint check_size_arg1(char *arg1)
{
  size_t size_arg1;
  
  size_arg1 = strlen(arg1);
  return (uint)(size_arg1 >> 3) & 1;
}
```

#### check_odd_bitwise_and.c

The next check looks at four different characters in the argument string to see if the last bit of their ascii code is a 0. An ascii character that satisfies this requirement is simply ascii "0".
```
int check_odd_bitwise_and(char *string)
{
  int return_code;
  
  if ((*string & 1U) == 0) {
    if ((string[1] & 1U) == 0) {
      if ((string[3] & 1U) == 0) {
        if ((string[5] & 1U) == 0) {
          return_code = 0;
        }
        else {
          return_code = 1;
        }
      }
      else {
        return_code = 1;
      }
    }
    else {
      return_code = 1;
    }
  }
  else {
    return_code = 1;
  }
  return return_code;
}
```

#### add_char_4_and_2.c

The next check calls a function that adds the third and fifth characters together.  The main function is checking if these equal the value for the ascii code of "m".  Two characters that satisfy this requirement are "0" and "=".
```
int add_char_4_and_2(char *string)
{
  return (uint)(byte)string[4] + (uint)(byte)string[2];
}
```

#### final_check.c

The last check of the argument string is checking if the seventh char is between the ascii characters quote and period.  A simple character that satisfies this is "#".  It is also checking if the eighth character is simply "0".
```
if (((argv[1][6] < '\"') || ('.' < argv[1][6])) || (argv[1][7] != '0')) {
```

#### compare.c

Once that set of checks is complete, the main function looks at the **flag.txt** file in the current directory.  It uses the function below to compare it to the static string.
```
void compare(char *param_1)
{
  strcmp(param_1,";]|^_j8d9YOZ;mrI]F=|:P5^iP7IPIKF4\\4IPI5AUb99\n");
  return;
}
```
One thing to note here is that the **flag.txt** provided had CRLF line endings when this check is for LF endings, so it needs to be converted using tr or a similar program.

#### decode_msg.c

Once all of these checks are satisfied the program will call the following function:
```
void decode_msg(void)
{
  printf("You can now decode the flag!");
  return;
}
```

In order to get this, I simply used the string "0000=0#0":
```
$ ./a.out '0000=0#0'
You can now decode the flag!
```

### Reversing something?

At this point, you might have noticed that we have an eight character string that doesn't really do much.  I attempted to use it to xor the flag, but unfortunately this got me nowhere.  It took me a lot of time at this point and a quick message to the challenge author to realize I needed to move onto some of the other functions in **a.out**.

Now, there were about ten functions in the file which were extremely similar only differing on one line.  These functions were not called anywhere else though.

#### encode.c

After a quick read through, I realized these functions were actually performing xor operations, so my guess was that one was used to encode the flag.

I spent some time going through one and renaming and retyping variables until it made sense as you can see below:
```
void encode(char *flag,char *key)
{
  int len_first_half;
  int len_second_half;
  size_t length;
  char *first_half;
  char *second_half;
  int k;
  int j;
  int i;
  char current_char_first_half;
  char current_char;
  
  i = 0;
  while( true ) {
    length = strlen(flag);
    if (length <= (ulong)(long)i) break;
    current_char = flag[i];
    flag[i] = flag[(long)i + 1];
    flag[(long)i + 1] = current_char;
    i = i + 2;
  }
  puts(flag);
  length = strlen(flag);
  len_first_half = (int)length / 2;
  len_second_half = (int)length - len_first_half;
  first_half = (char *)malloc((long)(len_first_half + 1));
  memcpy(first_half,flag,(long)len_first_half);
  first_half[len_first_half] = '\0';
  second_half = (char *)malloc((long)(len_second_half + 1));
  memcpy(second_half,flag + len_first_half,(long)len_second_half);
  second_half[len_second_half] = '\0';
  j = 0;
  while( true ) {
    length = strlen(first_half);
    if (length <= (ulong)(long)j) break;
    current_char_first_half = first_half[j];
    length = strlen(key);
    first_half[j] = current_char_first_half ^ (byte)length;
    j = j + 1;
  }
  k = 0;
  while( true ) {
    length = strlen(second_half);
    if (length <= (ulong)(long)k) break;
    second_half[k] = second_half[k] ^ key[4] + key[2] + 0x97U;
    k = k + 1;
  }
  first_half = strcat(first_half,second_half);
  printf("%s",first_half);
  return;
}
```

This is a lot to digest, but let's start at the beginning.  

#### char_swap.c

This code is simply iterating over the first string argument to the function, stepping two characters at a time.  It then swaps the characters so "abab" becomes "baba".

```
i = 0;
while( true ) {
  length = strlen(flag);
  if (length <= (ulong)(long)i) break;
  current_char = flag[i];
  flag[i] = flag[(long)i + 1];
  flag[(long)i + 1] = current_char;
  i = i + 2;
}
```

#### string_split.c

The following code snippet looks really complicated and confusing, but it's literally just splitting the flag in half into two strings.
```
length = strlen(flag);
len_first_half = (int)length / 2;
len_second_half = (int)length - len_first_half;
first_half = (char *)malloc((long)(len_first_half + 1));
memcpy(first_half,flag,(long)len_first_half);
first_half[len_first_half] = '\0';
second_half = (char *)malloc((long)(len_second_half + 1));
memcpy(second_half,flag + len_first_half,(long)len_second_half);
second_half[len_second_half] = '\0';
```

#### first_half.c

The first following while loop operates on the first half of the flag.  It simply xors each character with the length of the second string argument to the function.  Based on the code in the main function, I just assumed this length to be eight (mainly I took that the last index checked was 7 which means the key only cares about the first eight characters).
```
j = 0;
while( true ) {
  length = strlen(first_half);
  if (length <= (ulong)(long)j) break;
  current_char_first_half = first_half[j];
  length = strlen(key);
  first_half[j] = current_char_first_half ^ (byte)length;
  j = j + 1;
}
```

#### second_half.c

The next while loop works with the second half of the flag.  It xors each character with the sum of the third and fifth characters in the key combined with "+ 0x97U".  This is actually not the correct formula... Or it is... Ghidra is really confusing here, but if you look at the actual assembly, it turns out what is actually happening here is that insted of adding 0x97, the program is subtracting 0x69 from this value.  It is ultimately a lesson in that Ghidra is not always correct in how it decompiles things.
```
k = 0;
while( true ) {
  length = strlen(second_half);
  if (length <= (ulong)(long)k) break;
  second_half[k] = second_half[k] ^ key[4] + key[2] + 0x97U;
  k = k + 1;
}
```
This is also where each function in the **a.out** file differ.  Each function had a different value for which characters from the key are used to xor the second half of the flag.  But if you remember from main, it is checking if the third and fifth (index 2 and 4) characters add up to ascii "m", so my guess here is that this is the correct value for the xor.

The value for the xor here is actually static.  From main, we know that these two characters add up to ascii "m" and then just subtract 0x69 to get "0x4" or literally 4 in base 10.

#### last_bit.c

The last little snippet of this function is just concatenating the two halves of the flag again and printing it.
```
first_half = strcat(first_half,second_half);
printf("%s",first_half);
return;
```

## solve.py

So now, armed with the information that the flag has every character flipped, and that the first half is xor'd with 8 and the second half is xor'd with 4, we can decode the flag!

I wrote the following little python script to do it.  It just opens the flag file, swaps the characters back and then performs the correct opposite xor operation on each half of the flag.

```
#!/usr/bin/env python3
from math import floor
import base64

flag_txt_1 = open('flag.txt', 'r').read()

flag_txt = ''
for x in range(1,len(flag_txt_1),2):
    flag_txt += flag_txt_1[x]
    flag_txt += flag_txt_1[x-1]

key = '0000=0#0'
pairs = (2,4)

length = floor(len(flag_txt) / 2)
flag = ''
for x in range(0, length):
    flag += chr(ord(flag_txt[x]) ^ len(key)) # literally 8
for x in range(length, len(flag_txt)):
    flag += chr(ord(flag_txt[x]) ^ ord(key[pairs[0]]) + ord(key[pairs[1]]) - 0x69) # literally 4

print(base64.b64decode(flag).decode())
```

When done you can run the script and you will get the flag:
```
$ ./solve.py
SummitCTF{p35Ky_fuNc710N_C4115}
```

I enjoyed this challenge even though I wasn't able to solve it before the CTF ended.  Looking back on it now, it was a lot simpler than I made it, but I'll take it as a learning opportunity and try to not overcomplicate similar problems in the future.

Thanks for reading!