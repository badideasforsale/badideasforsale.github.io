---
layout: post
title:  "Using Secretive for Git"
date: 2023-11-03  10:30:00 -0700
categories: git secretive computers
---

# Using Secretive for Git
[Secretive](https://github.com/maxgoedjen/secretive) is a macOS app that allows you to store your SSH keys in the Secure Enclave. This is a Good Thing™, because it means that your private keys are never stored on disk, as is typical back in the olden days. Like yesterday. Instead, they're stored in the [Secure Enclave](https://support.apple.com/guide/security/secure-enclave-sec59b0b31ff/web), designed to be a place to put secrets.

It's easy to use, but so is putting a key in your `~/.ssh` folder.

## Why do security?
It's arguably easier to use tools like Secretive, with all the security benefits, than it is to do things any other way. So don't do it because it's secure, but because it makes your life better _and_ secure.

## Secretive setup
Once you've got Secretive installed, you need to generate a key. Instead of looking up the correct `ssh-keygen` incantation, Secretive will do it for you. Click the button, give it a name, and you're done. Technically, you can pick between have the key be unlocked whenever the computer is, or be promted each time the key is used. It's so quick and easy to touch your finger to the sensor, there's not much point to *not* doing so.

## GitHub setup
You next need to add the public key to GitHub. This is the same as any other key, except that you need to copy it from Secretive. Click the key, click the copy button, and paste it into GitHub. Done.

## Local setup
You will want to add an IdentityAgent section to your `~/.ssh/config` file. To quote the [man page](https://www.man7.org/linux/man-pages/man5/ssh_config.5.html):
> Specifies the UNIX-domain socket used to communicate with the authentication agent.
But you'll also want to add a ControlMaster and ControlPersist section to multiplex sessions. I use a 15 minute timeout, so after authenticating the key can be used for a short time without being prompted to re-authenticate.

```
Host *
        IdentityAgent /Users/_your_username_goes_here_/Library/Containers/com.maxgoedjen.Secretive.SecretAgent/Data/socket.ssh
        ControlMaster auto
        ControlPersist 15m
```

## Threads
There's one caveat to this setup, which is if a command runs that is multi-threaded. `brew update` is the biggest culprit. It will run multiple threads, and each thread will try to use the key, and if the key isn't already unlocked you will be prompted once-per-thread. The easiest way to handle this is to run some command to unlock the SSH key, then run `brew update`.