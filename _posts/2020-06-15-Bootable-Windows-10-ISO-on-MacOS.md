---
layout: post
title:  Creating a custom bootable Windows 10 ISO on a Mac
date:   2020-06-15 21:00:00 +0100
category: macos
tags: ["Windows", "Windows 10", "setup", "iso", "dvd", "install", "Mac", "MacOS"]
---

While writing my [previous post]({% post_url 2020-06-15-Windows-10-USB-on-MacOS %}) on creating a bootable Windows 10 USB stick on MacOS, I wanted to grab a screenshot of the setup error message that appears when `install.wim` is not where Windows Setup expects.

![Windows could not collect information for OSImage because the specified image file install.wim does not exist](/assets/posts/2020-06-15-Windows-10-USB-on-MacOS/windows_setup_error.png)

I couldn't find a way for VirtualBox to boot from a USB storage device, so figured I'd just extract the original ISO, delete the `install.wim` and build a new ISO image.  I'd then be able to boot this in a VirtualBox VM and get myself a screengrab of the error dialog.

Creating a bootable Windows 10 ISO from the contents extracted from the official ISO image took a fair amount of trial-and-error.  I finally settled on the following which seemed to work.

1.    Extract the contents of a Windows 10 ISO to a suitable location, let's call it `~/windows_10_custom`

      This can be done by mounting the ISO image and copying the contents using Finder, `cp -prv` or `rsync`... whatever your file copying tool of choice happens to be today.

2.    Make whatever custom changes you need to make within `~/windows_10_custom`

      In my case I just needed to remove `install.wim` to reproduce the relevant error message.

      > You'll need to ensure you leave the `boot/etfsboot.com` file in place if you want the ISO image to be bootable.

3.    Build a new bootable ISO by running the following command

      ```
      hdiutil makehybrid -o ~/windows_10_custom.iso -udf -iso -eltorito-boot ~/windows_10_custom/boot/etfsboot.com -no-emul-boot -boot-load-size 8 ~/windows_10_custom
      ```

4.    The ISO image will be created at `~/windows_10_custom.iso`

Simple as that.  Well, it is when you know how.
