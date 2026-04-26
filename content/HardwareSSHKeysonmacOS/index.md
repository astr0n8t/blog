---
title: "Hardware SSH Keys on macOS"
date: 2023-06-11T13:00:54-05:00
draft: false
author: "Nathan Higley"
authorLink: ""
description: "Utilize Hardware SSH Keys on macOS"

tags: [SSH, Security]
categories: [macOS, Yubikey]
---

Last year, I wrote a blog post about hardware SSH keys on Windows and WSL.  This year, I am running macOS full time, and have recently dealt with similar issues, and again had difficulty finding information on this topic on the internet.

Most of the solutions I've seen involve creating custom launchd scripts or similar and seem like a bit of a hacky solution.  While this solution I am about to present is also slightly hacky, it definitely feels more streamlined to use than any other solution I've seen and even includes support for saving the pin in Apple Keychain!

So without any further fluff, lets get to it.

## The Problem

Let me first frame the problem so you understand why I am writing this post.

### macOS SSH

When you use SSH on a stock brand new install of macOS, you may think that hardware SSH keys of the type "-sk" are supported out of the box.

Running ssh -V shows the following:
```
$ /usr/bin/ssh -V
OpenSSH_9.0p1, LibreSSL 3.3.6
```

This is a relatively recent build of OpenSSH. You would think it supported these types of SSH keys, and while it should, Apple chose to build it without support for them for whatever reason.

If you try to add one to the built in ssh-agent, you might see the following:
```
$ ssh-add ~/.ssh/id_ed25519_sk
Could not add identity "/Users/astr0n8t/.ssh/id_ed25519_sk": agent refused operation
```

And from the ssh-agent logs you will see:
```
process_add_identity: parse: unknown or unsupported key type
```

So this is the sad state of OpenSSH by default on macOS.  Hardware key types are strictly not supported out of the box.

### Homebrew to the Rescue

It honestly annoyed me that the keys aren't supported out of the box, but thankfully, homebrew will help us out here.

Just do the following to get actual OpenSSH:
```
brew install openssh 
```

This will install the binaries to `/usr/local/bin` instead of `/usr/bin`.

Checking the version gives you the following:
```
$ /usr/local/bin/ssh -V
OpenSSH_9.3p1, OpenSSL 1.1.1u  30 May 2023
```

Now, lets check how hardware SSH keys work with this version.

First, disable the built in ssh-agent:
```
 launchctl disable user/$UID/com.openssh.ssh-agent
```

Second, start the OpenSSH ssh-agent:
```
$ /usr/local/bin/ssh-agent -D
```

The '-D' flag tells ssh-agent not to fork so we can see the output.  Copy the first line which sets the `SSH_AUTH_SOCK` variable and paste it into another terminal.

In that terminal, now try to add your key:
```
$ ssh-add ~/.ssh/id_ed25519_sk
Identity added: /Users/astr0n8t/.ssh/id_ed25519_sk (ssh:)
```

It works!  And now, lets try to actually use the key in some method:
```
 ssh host
sign_and_send_pubkey: signing failed for ED25519-SK "/Users/astr0n8t/.ssh/id_ed25519_sk" from agent: agent refused operation
host's password: 
```

Hmm, that's not too good.  If we look at the logs for ssh-agent, we can see why:
```
Confirm user presence for key ED25519-SK SHA256:S4Hedoz6ovjA4g5Kg1RyLAEDF0g6gahFRrnEPkN5RXw
process_sign_request2: sshkey_sign: incorrect passphrase supplied to decrypt private key
```

Essentially, ssh-agent does not know how to ask us for our pin.

## Quick Solution without ssh-agent

There is a simple solution actually, if you just want a quick and dirty way to get your key working.

Kill the ssh-agent process and try again:
```
$ ssh host
Confirm user presence for key ED25519-SK SHA256:S4Hedoz6ovjA4g5Kg1RyLAEDF0g6gahFRrnEPkN5RXw
Enter PIN for ED25519-SK key /Users/astr0n8t/.ssh/id_ed25519_sk:
Confirm user presence for key ED25519-SK SHA256:S4Hedoz6ovjA4g5Kg1RyLAEDF0g6gahFRrnEPkN5RXw
User presence confirmed
```

Hey that worked!

### Caveats

Now, this does work, and I actually was using this for many months with no issues.  

The main caveat is that you have to either specify the key with `-i` or name it one of the standard key names such as `id_ed25519_sk`.

If you are happy with that, its a simple solution that should just work with the OpenSSH homebrew.  If you want a proper ssh-agent, keep reading.

## Better Solution with ssh-agent

Recently though, I wanted to be able to use a proper ssh-agent so I could forward it over SSH and for devcontainers.

Getting this to work actually took quite a bit of learning and understanding what is happening here.

First things first, you want to actually start ssh-agent.  I do this by enabling the `ssh-agent` plugin in oh-my-zsh.  You can also do this by starting it in your profile script.

Now essentially, ssh-agent does not know two things.  The first is it does not know *how* to ask us for our pin.  The second is it does not know *where* to ask us for our pin.

### The How 

The how is actually pretty simple, and you might have already stumbled upon a similar solution on the internet.

ssh-agent relies on the `SSH_ASKPASS` variable to know what program to call to get a password from a user when it is not running within a TTY.

If you install a ssh-askpass program and set this variable, ssh-agent will now know *how* to ask you for your pin.   (This will work if you run it with `-D`, but it will prompt you in the terminal session where ssh-agent is running, not SSH itself)

### The Where

The last bit of the puzzle is telling ssh-agent where to ask.  Now, ssh-agent by default is not run with the '-D' flag.  This means that it forks, and runs in the background, without a proper TTY.

Because of this, it might not be able to send you the request for your pin because it doesn't know where to send it.

The solution is to use a graphical ssh-askpass program, and to set the `DISPLAY` environment variable.  The `DISPLAY` environment variable tells ssh-agent that it can simply launch the program specified by the `SSH-ASKPASS` variable, and the program will be displayed to the user.

If you want a quick ask-pass program, you can use this one on GitHub: https://github.com/theseal/ssh-askpass which is fairly basic.

Once you install that, and set the following environment variables:
```
export SSH_ASKPASS=/usr/local/bin/ssh-askpass
export DISPLAY=":0"
```

If you are using oh-my-zsh make sure to place these lines before your zshrc sources oh-my-zsh so ssh-agent gets called with them.

You will get the following prompt regardless of how you started ssh-agent (as long as it sees the environment variables):
![prompt](./ssh-askpass.png)

So once again, this is a valid solution, and you can be satisfied with it.

## Custom Solution with ssh-agent and Keychain Support

At this point, I started to wonder, what does a ssh-askpass program actually entail.  It turns out, it just needs to print the password when called.  Nothing fancy.

I then decided that I wanted to find a better tool for the job then this ssh-askpass program.  I then stumbled accross pinentry.

### pinentry

The pinentry suite is a collection of tools that are written for gpg.  All they do is ask the user for their pin which seemed to fit since that is all I needed as well.

The source can be found here https://github.com/GPGTools/pinentry on GitHub.  It turns out that the macOS variant also supports storing the pin in Apple Keychain, and on top of that, this person wrote a version which supports using your fingerprint to unlock it as well: https://github.com/jorgelbg/pinentry-touchid.

I found this pretty cool, but more importantly, I figured these projects are better supported than the ssh-askpass one.

You can install pinentry-mac with brew:
```
brew install pinentry-mac
```

So I set off to write my own wrapper around this application, which actually wasn't too difficult.

I found this excellent blog post which helped me understand the protocol that these programs use: https://velvetcache.org/2023/03/26/a-peek-inside-pinentry/

I was then able to discern that if you simply send the "GETPIN" command to the program, it would prompt for the pin and return it.

The output returned needs to be filtered a bit, but that is simple enough with some Google and bashfu.

At this point, I had created a nice script that worked reliably, but it was lacking the ability to save the pin to keychain.

### Final Solution

Researching online gave me that there were two commands needed to enable support for Keychain, and that it had been disabled by default since it really shouldn't be enabled unless the individual wants it to be:
```
defaults write org.gpgtools.common UseKeychain -bool yes
defaults write org.gpgtools.common DisableKeychain -bool no
```

These should enable you to select to save the pin in Keychain, but I still did not see the option.

After about an hour of perusing the source code of pinentry-mac, I finally discovered why.  If I was actually using GPG with this utility, I would've seen the option, but the option relies on the KEYINFO about the key that the passphrase is requested for in order to identify the key later.

This makes sense.  The program needs a KEYID to know what to name and reference the pin by later.  Thankfully, from the previously mentioned blog post, I was able to find the command `SETKEYINFO` and `OPTION allow-external-password-cache` which would allow it to be saved in Keychain.

I had to do some string manipulation to send the SHA256 of the key with the `SETKEYINFO` command, but then it just works.

We get the following prompt now:

![prompt with keychain](./custom-ssh-askpass.png)

And selecting to save in keychain will create a new `GnuPG` item in our keychain with our key's pin.

Using that, now when we go to use our key, all we need to do is tap the key itself (assuming you have required both pin and presence when creating the resident SSH key).

The full ssh-askpass script can be found below:

```bash
#!/bin/bash
#
# Can enable check to enable keychain
# I've disabled this by default
# It only needs to run once
CHECK_KEYCHAIN_ENABLE=1
if [ $CHECK_KEYCHAIN_ENABLE -eq 0 ]
then
    USE_KEYCHAIN=$(defaults read org.gpgtools.common UseKeychain)
    if [ $USE_KEYCHAIN -eq 0 ]
    then
        defaults write org.gpgtools.common UseKeychain -bool yes
    fi
    DISABLE_KEYCHAIN=$(defaults read org.gpgtools.common DisableKeychain)
    if [ $DISABLE_KEYCHAIN -eq 1 ]
    then
        defaults write org.gpgtools.common DisableKeychain -bool no
    fi
fi

# We want to ignore confirmations for user presence
if [[ "$1" == "Confirm user presence"* ]]
then
    echo
else
    # See if we can get the hash of the key
    # that we want the password for
    # (this enables keychain option support)
    HASHTYPE=$(echo $1 | awk -F':' '{print $1}')
    if [[ "$HASHTYPE" == *"SHA256" ]]
    then
        # Grab the actual hash
        SHA256=$(echo $1 | awk -F':' '{print $2}')
        PROMPT="SETDESC $1\nOPTION allow-external-password-cache\nSETKEYINFO $SHA256\nGETPIN"
    else
        # Otherwise don't include the keyinfo
        PROMPT="SETDESC $1\nGETPIN"
    fi

    # Prompt the user for their pin
    PIN=$(echo -e $PROMPT | pinentry-mac | grep D | tr -d '\n')
    # Return the pin to ssh-agent starting after 'D '
    echo "${PIN:2}"
fi
```

Simply save this somewhere, make it executable, and set the `SSH_ASKPASS` variable to point to it.

And a bonus, you can edit and customize this script to your heart's content.

Enjoy, and feel free to leave a comment or suggestion on how to improve the script!
