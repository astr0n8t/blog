---
title: "NixOS: A Flaky Experience"
subtitle: "A noob's opinion"
date: 2024-04-20T13:57:45-05:00
draft: false
author: "Nathan Higley"
authorLink: ""
description: ""

tags: [linux, homelab]
categories: [nix, nixos, git]

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

I should preface this by saying that this will probably come off as a rant.

## Story Time

So once upon a time, I decided that I wanted to convert my homelab Kubernetes cluster to using NixOS as the base operating system.  The reason for this was simple: I'm already using GitOps via FluxCD for the actual configuration of the cluster, using NFS for the PVCs within the cluster, so the only thing not really documented in git was the nodes themselves.  Now normally with Kubernetes that's not much of an issue, you typically just deploy to cloud ephemeral nodes and then most of the complicated stuff is handled on the backend.  In a homelab though, everything is on-prem, and the real cloud is the lessons you learn along the way.

But NixOS seemed like a chance to change this, the whole operating system defined by a config file.  I had some pre-conceived notions about how I figured NixOS would work.  That is probably the real lesson in this: I took knowledge of how technologies like FluxCD worked and figured that another technology, i.e. NixOS, would work in a similar fashion.  The reality of the matter is that NixOS is more akin to a normal Linux distribution than I thought... But you can change this, and I can explain how I did it.

### What I Wanted

I should probably start by describing what I expected and wanted out of NixOS.  I wanted a way to turn a configuration defined in a Git repository into a disk image that the computer could boot from without any other changes and then anytime I pushed changes to the Git repository, the computer would apply that configuration.  This is similar to how FluxCD works, you create a configuration, run a quick `flux bootstrap` command, and then the cluster tracks the repository.

### The Reality

How NixOS works by default cannot accomplish this.  You run the installer similar to how you install any other distribution like Ubuntu or Fedora, and it generates a config file for you along with some hardware specific bits.  You then edit this config to add things to the OS, but its not really tracked anywhere by default, nor is there an easy way to pull that from anywhere.

## The Flexibility of nix

The neat thing about NixOS is also the really obnoxious part about it: it's built on top of the nix programming language which is a ["domain-specific, purely functional, lazily evaluated, dynamically typed programming language."](https://nix.dev/tutorials/nix-language.html)  What that means is that you can do basically anything as long as you can program it in nix.  And all of this revolves around nix the package manager.  So the functionality is there; just waiting for someone to program and build it.

### Feature Overload

Now I must address my real issue with NixOS: "experimental features."  Almost any blog post you read about NixOS tells you to enable Nix Flakes.  But as soon as you go to the NixOS wiki it tells you: ["Nix flakes is an experimental feature of the Nix package manager."](https://nixos.wiki/wiki/Flakes)  And then clicking the link for "experimental features" tells you from the NixOS manual ["Experimental features are considered unstable, which means that they can be changed or removed at any time."](https://nixos.org/manual/nix/stable/contributing/experimental-features.html)

But everywhere you go it tells you to enable flakes, and everyone that uses flakes will defend them.  To me this just seems like nix/NixOS doesn't want to embrace their own tools. The real issue I have with this is that it makes the documentation really hard to follow because to do what I was trying to accomplish I was forced to use flakes, but the people that write the documentation don't really seem to want to promote flakes although the rest of the internet does.

I understand aversion to change, but when the whole of the internet uses your ecosystem in your "experimental" way, at some point you need to accept it's no longer experimental and embrace/support it as well for the betterment of your project. Now back to the story.

### Finding the solution

I stumbled my way through the pages of Google to [nixos-generators](https://github.com/nix-community/nixos-generators/): a project that allows you to take a NixOS config and generate a bootable disk image (or installer?).  This seemed almost perfect to what I needed minus the syncing with the git repository.  But this project also presented with a weird issue: after you generate the image, you could no longer apply updates to your configuration via `nixos-rebuild switch`... Which is the only way to update the configuration.

At this point, I was really confused.  It almost seemed like what I wanted didn't exist.  I really didn't want to have to take my servers and plug them into a monitor and keyboard or SSH into an installer to finish installing them.  I just wanted to plug the SSD into my computer run `dd`, plug it back into my server, and have it boot to known good.  In fact, I need this as I constantly break things and don't want to waste time re-installing the OS when it can be done automatically.

Then I found an issue on the nixos-generators website that told me what I wanted to see: [link to the issue](https://github.com/nix-community/nixos-generators/issues/193#issuecomment-1937095713).  This comment by GitHub user [@JustinLex](https://github.com/JustinLex) described having two outputs of a nix flake, one for the generator and one for the OS configuration.  You then had to make a custom file that described the boot disks and add it to your OS configuration so that the bootloader was actually setup properly on a `nixos-rebuild switch` as well as add a `system.autoUpgrade` block to your configuration that updated from your flake.

Now I had what I needed!  And this highlights a huge win for NixOS: the community is amazing and finds ways to make things work.  It just confuses me how the community can be this awesome and simultaneously the project doesn't seem to have good support or documentation.

## Managing Secrets 

So now I had a way to generate a disk image from a git repository and then the system will auto-apply the repository configuration weekly.  Perfect?  Almost. 

The last thing I needed was a way to inject secrets into the image.  The reason for this is 1) I wanted my git repository to be private and 2) I needed a way to add the k3s cluster token to all three nodes.

### The NixOS Way (TM)

It seems like the NixOS way is documented in [this page](https://nixos.wiki/wiki/Comparison_of_secret_managing_schemes).  Basically, most of the community seems to want to use either [agenix](https://nixos.wiki/wiki/Agenix) or [sops-nix](https://github.com/Mic92/sops-nix) and while these seemed like fine ideas, they were opposite of what I wanted for my project.

The first issue is that I don't want to store encrypted blobs in my git repository.  I know its probably secure, but I'd just rather not if I can help it when better solutions exist like a password manager or secrets manager.

The second issue is related to the first: if you are going to encrypt the secrets, that necessitates decrypting them somehow.  Which means somehow you have to have a secret which is either generated per-machine (i.e. a manual process) or shared via symmetric key (i.e. another secret that cannot be encrypted itself and we begin recursion here?).

### What I Wanted

All I really wanted was a way to create files with secrets on the generated disk image that were only readable by root.  They wouldn't be stored in the git repository, and I would only need to inject them once in some automated form.  Now, you might argue that its bad to have plaintext secrets on disk, and to that I would argue that unless you're storing the encryption keys in the TPM of the computer or some hardware module, there's not much difference to having encrypted secrets on disk because they have to be decrypted at some point and the encryption key has to also be on disk in some form at that point.

### The Issue

My first attempt at this was to simply see if I could somehow create the files in the configuration passed to nixos-generators and then it would just be there.  I accomplished this after a lot of trial and error with a weird shell script and `readFile` calls in the nix configuration.  The first issue with this is that a) its stored in the nix store which is world readable for some reason and b) when you run `nixos-rebuild` it just deletes the files since it's not present in the OS configuration.

So then I added `readFile` calls in the OS configuration pointing to the files I created via the nixos-generators configuration.  But this led to an issue: nix flakes have to be able to be built without outside resources and doing it this way was considered "unpure".

Now I could have just lived with an impure flake, but I also wasn't super happy about my janky shell script system.

### My Solution

What I ended up realizing was that since I could generate a disk image, I could also mount the disk image and create any files I wanted on it and treat it like a normal system.  So my solution was actually to just use Ansible in conjunction with local actions to create files on the disk.  This seemed like a great solution for me because I can also re-use the playbook after the systems are stood up and rotate my secrets with minimal effort.

With the secrets on disk, you just have to use the corresponding file directives in the nix configuration such as `hashedPasswordFile`.

## Putting it All Together

The last piece of the puzzle is that I wanted to decouple the functions that create the specific configurations for nixos-generators and the OS configurations.  There really was no reason for this other than I wanted a way to re-use the functionality in a different flake in the future.  This led me to try to figure out how flakes and subflakes worked, which led to a new issue: I'm really out of practice in functional programming.

So by lots of trial and error and hard lessons with cryptic errors such as "error: cannot coerce a set to a string" and trying to figure out how list and set merging actually worked, I created [nixos-gitops](https://github.com/astr0n8t/nixos-gitops/) which makes it simpler to do what I did.

To wrap it all up, I also was able to enable [Mend Renovate](https://www.mend.io/renovate/) on the repository to update the flake.lock file, create a pull request, and thus have a way to weekly upgrade my nodes with a manual review process.  This also enables you to quickly rollback to a known good, and if for some reason the nodes are completely trashed, I just have to re-image a few disks and then everything is fine again.

### My Thoughts

I really do like my new setup.  NixOS is definitely an upgrade to manually configuring Linux systems and in the future I'll probably start using it on my host system as well.  Once you get a setup working, it continues to work and seems to be rock solid and minimal, similar to a container runtime, which I like.

### My Complaints

The project maintainers need to embrace the way that their userbase is actually using their project.  What I mean by that is build the relevant documentation that will present users with the best/modern way to use nix/NixOS.  

Also, I would love if the project evolved to have the functionality of nixos-generators built in.  Why would you need an installer for an OS that runs off of a configuration file when you already have the configuration file?  I think the reason they do not currently do this is that they market the OS for personal workstation use, but I think they are missing a huge market share by doing this.  Being able to define configurations that generate golden images for servers and workstations or VDI is amazing.  Imagine if you could take your NixOS flake in a git repo and simply do `nix image build --flake <repo> --out disk.img` and you're done.  You would not need any of this weird bandaid flake code I had to write to get the OS and the generator flake to work together.

## In Closing

I don't want anyone to get the wrong idea about my feelings towards NixOS: I absolutely love the technology.  I just think it could be so much more and be a better experience for end users.  

Rant over.  Thank you if you actually read the post and didn't just rush to the comments to call me a noob (although you wouldn't be wrong).

