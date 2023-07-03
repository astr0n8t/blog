---
title: "Tornado Writeup"
subtitle: "UIUCTF 2023"
date: 2023-07-03T08:45:00-05:00
lastmod: 2023-07-03T08:45:00-05:00
draft: false
author: "Nathan Higley"
authorLink: ""
description: ""

tags: [ctf, writeup]
categories: [Writeups, UIUCTFCTF 2023]

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

Tornado was a misc challenge that involved a singular wav file with the hint that it was encoded using SAME encoding. If you want to learn more about this encoding, you can read some more [here](https://en.wikipedia.org/wiki/Specific_Area_Message_Encoding), but essentially it boils down to being the way that emergency weather alerts are sent within the US (and other some other countries as well).

Upon doing some research on how to actually read this file, I came across [sameold](https://github.com/cbs228/sameold) which contains a SAME decoder, samedec, written in rust, meant to be used with a Raspberry Pi or PC to receive these emergency alerts.  It took a little bit of trial and error to realize that there is a specific way to read in wav files:
```
sox 'Same.wav' -t raw -r 22.05k -e signed -b 16 -c 1 - | samedec -r 22050
```

I probably would've figured this out a lot sooner if I actually read the README.  Anyways, if you're on Kali (or probably most distros) you can install sox with a quick:
```
apt install sox
```

If you run samedec with no flags, it will just figure out the actual SAME message: 
```
ZCZC-WXR-TOR-018007+0045-0910115-KIND/NWS-
```
Which is not what we want. (Fun fact: the message is a tornado warning for Benton County Indiana from April 1st, effective from 1:15AM to 2:00AM UTC)

So instead, we add the **-v** flag to get more info and it will actually give us the bits we need to decode the flag:
```
/home/kali/CTF/UIUCTF23/misc/tornado [git::uiuctf23 *] [kali@kali] [18:54]
> sox warning.wav -t raw -r 22.5k -e signed -b 16 -c 1 - | ./samedec-x86_64-unknown-linux-gnu -r 22050 -v
 INFO  samedec > SAME decoder reading standard input
 INFO  sameold::receiver > [1400          ]: event [link]: searching
 INFO  sameold::receiver::framing > burst: started: after 20 bytes
 INFO  sameold::receiver          > [7992          ]: event [link]: reading
 INFO  sameold::receiver          > [22467         ]: event [link]: decoded burst: "ZCZC-UXU-TFR-R18007ST_45-0910BR5-KIND3RWS-"
 INFO  sameold::receiver          > [22467         ]: event [transport]: assembling
 INFO  sameold::receiver          > [22488         ]: event [link]: no carrier
 INFO  sameold::receiver          > [44369         ]: event [link]: searching
 INFO  sameold::receiver::framing > burst: started: after 19 bytes
 INFO  sameold::receiver          > [50614         ]: event [link]: reading
 INFO  sameold::receiver          > [65087         ]: event [link]: decoded burst: "ZCZC-WIR-TO{3018W0R+00T5-09UT115-K_EV/NWS-"
 INFO  sameold::receiver          > [65108         ]: event [link]: no carrier
 INFO  sameold::receiver          > [86644         ]: event [link]: searching
 INFO  sameold::receiver::framing > burst: started: after 20 bytes
 INFO  sameold::receiver          > [93236         ]: event [link]: reading
 INFO  sameold::receiver          > [107708        ]: event [link]: decoded burst: "ZCZC-WXRCTOR-0D_007+004OR_O1011E@KIND/N}S-"
 INFO  sameold::receiver          > [107729        ]: event [link]: no carrier
 INFO  sameold::receiver          > [136628        ]: event [transport]: message: (100.0% voting, 120 errors) "ZCZC-WXR-TOR-018007+0045-0910115-KIND/NWS-"
ZCZC-WXR-TOR-018007+0045-0910115-KIND/NWS-
 INFO  sameold::receiver          > [136671        ]: event [transport]: assembling
 INFO  sameold::receiver          > [347741        ]: event [transport]: idle
 INFO  sameold::receiver          > [2006035       ]: event [link]: searching
 INFO  sameold::receiver::framing > burst: started: after 20 bytes
 INFO  sameold::receiver          > [2012622       ]: event [link]: reading
 INFO  sameold::receiver          > [2013916       ]: event [link]: decoded burst: "NNNNfff"
 INFO  sameold::receiver          > [2013916       ]: event [transport]: message: (0.0% voting, 0 errors) "NNNN"
NNNN
 INFO  sameold::receiver          > [2013937       ]: event [link]: no carrier
 INFO  sameold::receiver          > [2013937       ]: event [transport]: assembling
 INFO  sameold::receiver          > [2035822       ]: event [link]: searching
 INFO  sameold::receiver::framing > burst: started: after 19 bytes
 INFO  sameold::receiver          > [2042061       ]: event [link]: reading
 INFO  sameold::receiver          > [2043355       ]: event [link]: decoded burst: "NNNNfff"
 INFO  sameold::receiver          > [2043377       ]: event [link]: no carrier
 INFO  sameold::receiver          > [2064914       ]: event [link]: searching
 INFO  sameold::receiver::framing > burst: started: after 20 bytes
 INFO  sameold::receiver          > [2071503       ]: event [link]: reading
 INFO  sameold::receiver          > [2072798       ]: event [link]: decoded burst: "NNNNfff"
 INFO  sameold::receiver          > [2072819       ]: event [link]: no carrier
```

That gives us three different messages:
```
ZCZC-UXU-TFR-R18007ST_45-0910BR5-KIND3RWS-
ZCZC-WIR-TO{3018W0R+00T5-09UT115-K_EV/NWS-
ZCZC-WXRCTOR-0D_007+004OR_O1011E@KIND/N}S-
```

If you look closely, you can probably already see the flag, but the trick is to grab the unique character at each position out of the three strings.

I wrote this quick python script to recover it:
```
a = "ZCZC-UXU-TFR-R18007ST_45-0910BR5-KIND3RWS-"
b = "ZCZC-WIR-TO{3018W0R+00T5-09UT115-K_EV/NWS-"
c = "ZCZC-WXRCTOR-0D_007+004OR_O1011E@KIND/N}S-"
d = ""

for x in range(len(a)):
    # Grab the character at this position from all three strings
    chars = [a[x], b[x], c[x]]
    # Convert the list to a set which will filter out duplicates
    # and then back to a list so we can use indices to access it
    uniq = list(set(chars))
    # If there is only one uniq character then add it
    if len(uniq) == 1:
        d += uniq[0]
        continue
    # Check how many times the first item in the list appears in the
    # three strings, if it appears once then its the unique character
    if chars.count(uniq[0]) == 1:
        d += uniq[0]
    # If it appears more than once than the other character is the unique one
    else:
        d += uniq[1]

print(d)
```

And we get the flag:
```
ZCZC-UIUCTF{3RD_W0RST_TOR_OUTBRE@K_EV3R}S-
```

I enjoyed this challenge and while the solution is simple to explain, it took quite a bit of time to find the correct tool and figure out how to actually get the flag.  Let me know in the comments if you found a better way to do it or if you have something to add!
