---
title: "Checkmate in 1"
subtitle: "MetaCTF CyberGames 2020"
date: 2020-10-26T11:32:23-04:00
lastmod: 2020-10-26T11:32:23-04:00
draft: false
draft: false
author: "Nathan Higley"
authorLink: ""
description: ""

tags: [ctf, writeup, crypto]
categories: [Writeups, MetaCTF 2020]

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

# Checkmate in 1

This challenge was interesting, because I actually found it relatively easy while my teammates could not figure it out earlier during the competition.  Keep in mind that this was probably the last challenge I solved, and I think I solved it around 3:30 - 4:00 AM on Sunday.  Anyways enough talk lets get into the challenge.

You can find a backup of the challenge [here on my Github](https://github.com/astr0n8t/MetaCTF2020/tree/main/Checkmatein1) just in case you don't have access to it.

The basics of the challenge is we had the string "F^mY;L?t24Zk.m^-hnWl,[l)[ku" and nine different img of chessboards.  The only other thing to note is that the flag is wrapped with MetaCTF{} which means that the first eight characters of the string is MetaCTF{.

## Solving the Chessboards

The first thing I needed to do was solve the actual chessboards, the first of which is pictured below:

![The first checkerboard](https://raw.githubusercontent.com/astr0n8t/MetaCTF2020/main/Checkmatein1/1.png)

The solution is how can you move a white piece to cause checkmate in a single move. 
I am not a chess player, but fortunately one of my teammates had been working on the challenge before and was able to provide me with the following solutions:

```
1: Queen from c2 to h7 
2: Rook from e1 to e8
3: Rook from c7 to e7
4: Pawn from d4 to d5
5: Bishop from f1 to b5
6: Knight from c4 to b6
7: Rook from a1 to a8
8: Rook from h2 to h8
9: Queen from f6 to h8
```

## Cracking the Cipher

Cool, now what do these mean.  Well given that the first eight letters of the string are MetaCTF{ I decided to start with that:
```
M (77) - F (70) = 7
e (101) - ^ (94) = 7
t (116) - m (109) = 7
a (97) - Y (89) = 8
C (67) - ; (59) = 8
T (84) - L (76) = 8
F (70) - ? (63) = 7
{ (123) - t (116) = 7
```

If you notice, I also provided the ascii character codes and their differences, notice anything?  I did.  I forgot to mention that while there are nine chessboards, there are also twenty-seven characters in the provided string.  I already had a suspicion at this point that it was a shift cipher, so doing some math I hypothesised that each chessboard was the shift for three characters.  If you notice the number in the solution to the first three chessboards corresponds to the shifts for the first eight characters.

Going off of that we can do the same thing with all twenty-seven characters I constructed the following solution:

```
M (77) - F (70) = 7
e (101) - ^ (94) = 7
t (116) - m (109) = 7
a (97) - Y (89) = 8
C (67) - ; (59) = 8
T (84) - L (76) = 8
F (70) - ? (63) = 7
{ (123) - t (116) = 7
9 (57) - 2 (50) = 7
9 (57) - 4 (52) = 5
_ (95) - Z (90) = 5
p (112) - k (107) = 5
3 (51) - . (46) = 5
r (114) - m (109) = 5
c (99) - ^ (94) = 5
3 (51)- - (45) = 6
n (110) - h (104) = 6
t (116) - n (110) = 6
_ (95) - W (87) = 8
t (116) - l (108) = 8
4 (52) - , (44) = 8
c (99) - [ (91) = 8
t (116) - l (108) = 8
1 (49) - ) (41) = 8
c (99) - [ (91) = 8
s (115) - k (107) = 8
} (125) - u (117) = 8
```

Since this was at 4:00 AM, I made a few mistakes initially mathwise, but eventually did solve it to get the following flag:
>MetaCTF{99_p3rc3nt_t4ct1cs} 


