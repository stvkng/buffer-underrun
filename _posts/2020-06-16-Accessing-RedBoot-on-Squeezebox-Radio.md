---
layout: post
title:  Accessing RedBoot on a Squeezebox Radio
date:   2020-06-16 09:00:00 +0100
category: squeezebox
tags: ["squeezebox", "radio", "logitech", "hardware", "redboot"]
---
![A Logitech Squeezebox Radio, in red](/assets/posts/2020-06-16-Accessing-RedBoot-on-Squeezebox-Radio/squeezebox_radio.png)

The current global pandemic has meant most of us are staying at home quite a bit more than usual.  Working from home full time might be a little lonely, but does mean saving a fair number of commuting hours each week.

With more time to fill, I decided to look into resurrecting my fleet of ageing [Logitech Squeezeboxes](https://en.wikipedia.org/wiki/Squeezebox_(network_music_player)).  These have been unused and cluttering up the house for some time.  They've long been abandoned by Logitech, but there remains an active community around them.  The server software and much of the firmware is open-source and available on GitHub.

Wanting to tinker with the firmware of the SB Radio (and probably eventually my Squeezebox Controller), one of the first jobs was to see if I could gain access to the firmware bootloader.  This would potentially provide a way to recover the situation if I managed to break things and turn one of my radios into a brick (a pretty likely scenario, let's be honest).

Connecting to the serial console on the radio (a topic for a future post, no doubt) reveals that it uses [RedBoot](https://en.wikipedia.org/wiki/RedBoot).

```
++NAND: RCSR=54200900
Searching for BBT table in the flash ...
.
Found version 1 Bbt0 at block 1023 (0x7fe0000)
Total bad blocks: 0
.FEC PHY: RTL8201EL
FEC: [ HALF_DUPLEX ] [ disconnected ] [ 10M bps ]:
Ethernet mxc_fec: MAC address [***redacted***]
No IP info for device!
Unrecognized chip: 0xf8!!!
hardware reset by POR

Clock input is 24 MHz
RedBoot(tm) bootstrap and debug environment [ROMRAM]
Non-certified release, version FSL 200904 - built 16:56:15, Jan  5 2012

Platform: Logitech Baby (i.MX25 )  PASS 1.0 [x32 DDR]
System type 2070 revision 7
Copyright (C) 2000, 2001, 2002, 2003, 2004 Red Hat, Inc.
Copyright (C) 2003, 2004, 2005, 2006 eCosCentric Limited

RAM: 0x00000000-0x03f00000, [0x00095250-0x03ef1000] available
FLASH: 0x00000000 - 0x8000000, 1024 blocks of 0x00020000 bytes each.
RedBoot> ubi attach -f 0x80000 -l 0x07EC0000 -w 0x800 -e 0x20000 -s 0x200
scanning 1014 PEBs
............................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................scanning is finished
RedBoot> ubi load -b 0x100000 kernel%{os_backup}
RedBoot> exec -c %{os_cmdline}entry=0x80008000, target=0x80008000
Using base address 0x00100000 and length 0x002d4800
[    0.000000] Linux version 2.6.26.8-rt16 (parabuild@vdc01b01centos01) (gcc version 4.4.1 (Sourcery G++ Lite 2010q1-202) ) #1 PREEMPT RT Mon Sep 12 21:59:07 MDT 2011
```

RedBoot appears to be a pretty common bootloader used on embedded devices.  It has a built-in command line interface that can manage the boot configuration and flash images.  Aha!  Hopefully this will offer a way to reflash some good firmware if the device becomes bricked at any point.

So, how do we get access to the RedBoot command line?  All Google searches on this topic said that we should be able to press `Ctrl-C` during the boot process to cancel the boot script and get a `RedBoot>` prompt.  I tried this many times, without success.  I even tried repeatedly stuffing `\003` into my `screen` serial session with no success.

Some sources suggested that the following message should be printed during the boot process, at which point we can hit `Ctrl-C` to gain access to RedBoot:

```
== Executing boot script in 10 seconds - enter ^C to abort
```

However, this isn't happening on the SB Radio; there is no such message or pause during boot.  We immediately drop into what I presume is the start of the boot script (`RedBoot> ubi attach...`) with no opportunity to cancel:

```
Platform: Logitech Baby (i.MX25 )  PASS 1.0 [x32 DDR]
System type 2070 revision 7
Copyright (C) 2000, 2001, 2002, 2003, 2004 Red Hat, Inc.
Copyright (C) 2003, 2004, 2005, 2006 eCosCentric Limited

RAM: 0x00000000-0x03f00000, [0x00095250-0x03ef1000] available
FLASH: 0x00000000 - 0x8000000, 1024 blocks of 0x00020000 bytes each.
RedBoot> ubi attach -f 0x80000 -l 0x07EC0000 -w 0x800 -e 0x20000 -s 0x200
scanning 1014 PEBs
```

I'm thinking now that perhaps Logitech set the boot script timeout to zero, essentially locking it.  Then I noticed the following paragraph in [this document](https://doc.ecoscentric.com/ref/redboot-commands-and-examples.html).

> RedBoot supports the notion of a boot script timeout, i.e. a period of time that RedBoot waits before executing the boot time script. This period is primarily to allow the possibility of canceling the script. Since a timeout value of zero (0) seconds would never allow the script to be aborted or canceled, this value is not allowed. If the timeout value is zero, then RedBoot will abort the script execution immediately.

So, if the boot script timeout is zero, it shouldn't get executed.  This is odd.  Maybe then the bootloader on the Squeezebox Radio has been locked down in some proprietary way.

I walked away at this point, a little disappointed, thinking that these devices would be pretty much unrecoverable in the event a firmware snafu.

A day or so later though, I couldn't let this go.  Eventually I stumbled upon a `bootloaders` source repository on the SlimDevices SVN: http://svn.slimdevices.com/repos/jive/7.8/trunk/bootloaders/  This is one of the Squeezebox source repos that has not made its way onto GitHub.  Lo and behold, there is a `redboot` directory.

![The bootloaders SVN repository](/assets/posts/2020-06-16-Accessing-RedBoot-on-Squeezebox-Radio/bootloaders_svn.png)

Perusing this repo yields some interesting nuggets, like [`squeezeos-redboot.patch`](http://svn.slimdevices.com/repos/jive/7.8/trunk/bootloaders/redboot/imx25/patches/logitech/squeezeos-redboot.patch).

```
+#ifdef SQUEEZEOS_REDBOOT
+    if (script) {
+	extern int squeezeos_halt_boot;
+
+	if (squeezeos_halt_boot) {
+	    script = NULL;
+	}
+    }
+    if (script && script_timeout > 0) {
+	/* A physical button is used to stop boot, allow a zero timeout */
+#else
     if (script) {
+#endif // SQUEEZEOS_REDBOOT
         // Give the guy a chance to abort any boot script
         unsigned char *hold_script = script;
         int script_timeout_ms = script_timeout * CYGNUM_REDBOOT_BOOT_SCRIPT_TIMEOUT_RESOLUTION;
```

It turns out that Logitech patched the RedBoot bootloader on the radio to allow a zero timeout value because "`a physical button is used to stop boot`".  Interesting.  Which button could it be?  There aren't many... brute force?  Not needed.  The secret is given away in [`baby-buttons.patch`](http://svn.slimdevices.com/repos/jive/7.8/trunk/bootloaders/redboot/imx25/patches/logitech/baby-buttons.patch):

```
+    /* Pause to halt boot */
+    writel(col_val | (0xD << 1), GPIO3_BASE_ADDR + GPIO_DR);
+    hal_delay_us(5);
+    val = readl(GPIO2_BASE_ADDR + GPIO_DR);
+    if ((val & (1 << 30)) == 0) {
+	diag_printf("== aborted\n");
+	squeezeos_halt_boot = true;
+	return;
+    }
```

The pause buttton... makes sense I guess.  Let's try it.  Holding the pause button while powering up the radio gives this:

```
++NAND: RCSR=54200900
Searching for BBT table in the flash ...
.
Found version 1 Bbt0 at block 1023 (0x7fe0000)
Total bad blocks: 0
.== aborted
FEC PHY: RTL8201EL
FEC: [ HALF_DUPLEX ] [ disconnected ] [ 10M bps ]:
... waiting for BOOTP information
Ethernet mxc_fec: MAC address [***redacted***]
Can't get BOOTP info for device!
Unrecognized chip: 0xf8!!!
hardware reset by POR

Clock input is 24 MHz
RedBoot(tm) bootstrap and debug environment [ROMRAM]
Non-certified release, version FSL 200904 - built 16:56:15, Jan  5 2012

Platform: Logitech Baby (i.MX25 )  PASS 1.0 [x32 DDR]
System type 2070 revision 7
Copyright (C) 2000, 2001, 2002, 2003, 2004 Red Hat, Inc.
Copyright (C) 2003, 2004, 2005, 2006 eCosCentric Limited

RAM: 0x00000000-0x03f00000, [0x00095250-0x03ef1000] available
FLASH: 0x00000000 - 0x8000000, 1024 blocks of 0x00020000 bytes each.
RedBoot>
```

Success!  We have a `RedBoot>` prompt!  We can type `help` and get a list of commands.

```
RedBoot> help
Setup/Display clock
Syntax:
   clock [<ARM core clock in MHz> [:<ARM-AHB clock divider>]
If a selection is zero or no divider is specified, the optimal divider values
will be chosen. Examples:
<...snip...>
```

Looks like this could well be useful then.

Note that the display on the radio remains blank when doing this.  Also, there is a long pause when `... waiting for BOOTP information` is displayed; it takes maybe around a minute to get past this, at least when not attached to a network.

This probably took me a few hours over a couple of days to figure out.  If only I'd waited a few weeks, I would've found the answer posted on the excellent Squeezebox forums [here](https://forums.slimdevices.com/showthread.php?112339-Squeezebox-Radio-start-up-loop/page2) where 'mrw' mentions that pressing the pause button interrupts the boot process and gives you a RedBoot prompt.

Oh well, it was fun figuring it out nevertheless.
