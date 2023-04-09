---
title: "[REDACTED]"
subtitle: "MetaCTF CyberGames 2020"
date: 2020-10-26T11:37:19-04:00
lastmod: 2020-10-26T11:37:19-04:00
draft: false
author: "Nathan Higley"
authorLink: ""
description: ""

tags: [ctf, writeup]
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
This will be a short but sweet writeup.  You can find a backup of the challenge [here on my GitHub](https://github.com/astr0n8t/MetaCTF2020/tree/main/REDACTED) in case you don't have access to it.

The challenge is simply to extract some data from a PDF file.

When you open the PDF you are presented with this:

![The PDF](redacted.png)

Knowing this, it looked like there is just an image or drawing over what we want to see, so I did some research to find a suitable program to disassemble the PDF.

I stumbled upon mutools which I installed in Kali:
```
pt install mupdf-tools
```

While the tutorial I was following had some complicated steps to get what I was looking for, I simply gave
```
mutool extract cybercorp_memo.pdf
```
This gave me the following image:

![The embedded image](redacted_embed.png)

And the follwing flag:
>MetaCTF{politics_are_for_puppets}

And that's it!  Just knowing the right tool for the job makes it pretty easy.
