# Thinkpad X1 Yoga Gen 3: Fix touchscreen after sleep

This document describes my foolish and noobish jouney to figuring out how to fix the fact that the touchscreen of my Thinkpad was not working after waking it from sleep.

[The Journey](#the-journey) was the fun (and sometimes frustrating part) but if you just want the solution you can skip to [The Destination](#the-destination)

---

# The Journey

## Arch Wiki

https://wiki.archlinux.org/title/Lenovo_ThinkPad_X1_Yoga_(Gen_3)#Using_acpi_call

---

### Fix touchscreen after resume

The X1Y3 has a firmware bug where the touchscreen may not come up again after waking up from S3 suspend/resume.

The following fixes were pulled from: Lenovo Linux Forums
Using acpi_call

#### Install and enable the acpi_call kernel module.

Add the following systemd service:

`/etc/systemd/system/activate-touch-hack.service`

```
[Unit]
Description=Touch wake Thinkpad X1 Yoga 3rd gen hack
After=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target

[Service]
ExecStart=/bin/sh -c "echo '\\_SB.PCI0.LPCB.EC._Q2A'  > /proc/acpi/call"

[Install]
WantedBy=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

Enable activate-touch-hack.service.

#### Disabling Thunderbolt

Some users have reported that disabling Thunderbolt in BIOS -> Security -> IO ports -> Thunderbolt permanently fixes the touchscreen issue. As a consequence, docking stations may have some features disabled. 

---

## acpi_call

Disabling Thunderbolt is not an option -> install acpi_call

Thankfully Arch points to a repository for acpi_call that is kept up to date with the current Linux kernel by the nix community:

https://github.com/nix-community/acpi_call/ is the current maintained fork of https://github.com/mkottman/acpi_call

### Installation

`git clone ...` -> `make`?

#### Problems

Same problems as u/egaleclass18 at https://www.reddit.com/r/Fedora/comments/vrsh3q/how_to_install_acpi_call/ 

```
...
... /lib/modules/5.18.9-200.fc36.x86_64/build: No such file or directory.  Stop.
...
```

#### Solution?

Provided by u/snmpenv:

```
sudo dnf install kernel-devel
will install the files that the makefile is looking for.
might also need kernel headers:
sudo dnf install kernel-headers
```

#### The build works, so make install now?

`sudo make install` actually...

Works, but `sudo modprobe acpi_call` does not work: Key was rejected by service.

The error was apparently caused by secure boot. I could just disable it, but we have been running from secure boot enough now. It's time that it catches up to us.

Luckily nix-community/acpi_call has a disclaimer about this:

#### Secure Boot

> **Notes on dkms and Secure Boot**

> If that is the way you want to install this module, you can follow 
[this guide](https://web.archive.org/web/20210215173902/https://gist.github.com/dop3j0e/2a9e2dddca982c4f679552fc1ebb18df) ([mirror](https://gist.github.com/s-h-a-d-o-w/53c2215e955c3326c6ec8f812a0d2f27)) to have dkms automatically sign the module after building.

So I installed a new machine owner key by running the `one-time-setup` script from the Gist. Very exciting.

It's basically just 2 commands: creating a key with openssl, and putting it in the machine owner key database using mokutil

To set up dkms to sign after building I was supposed to create a file in `/etc/dmks/`. That directory did not exist - very suspicious. So I did `dnf install dkms` to be sure.

##### A short detour through old workarounds

So I created the file like I was told:  `vim /etc/dkms/acpi_call.conf`: `POST_BUILD=../../../../../../root/module-signing/dkms-sign-module`.

After that `make` && `sudo make install` but still nothing.

But wait the Makefile has targets for dkms. So what about dkms? I just installed it

1. `sudo make dkms-add`
2. `sudo make dkms-build`
3. `sudo make dkms-install`
4. `sudo make modprobe-install`

Still the same error: Key was rejected by service.

#### Checking dkms conf

Maybe I did not configure the automatic signing correctly? Maybe I need to configure something? Lets check what else is in the `/etc/dkms/` directory:

In the `/etc/dmks/framework.conf` there is an option to set a signing key. Apparently that option was added in version 3.

While looking for more information I found this: https://gist.github.com/siddhpant/19c07b07d912811f5a4b2893ca706c99

A guide basically outlining that I just need to set these to options and it should just work. I already generated a key earlier using the `one-time-setup` and added it to the MOK DB, so lets use that.

Unfortunately I gave the key a password. That password would be making things more complicated, so I removed it:

1. `mv MOK.priv MOK_pw.priv`
2. `openssl rsa -in MOK_pw.priv -out MOK.priv`

Now lets set them in the config and try again:

1. `vim /etc/dkms/framework.conf`, add lines
2. `mok_signing_key="/root/module_signing/MOK.priv"` and
3. `mok_certificate="/root/module_signing/MOK.der"`

#### So lets build one final time

1. `sudo make dkms-add`
2. `sudo make dkms-build`
3. `sudo make dkms-install`
4. `sudo make modprobe-install`

No errors, yibbie

## The final test

Finally coming back to the fix from the Arch wiki, having installed the required kernel module

1. Close Laptop
2. Repoen Laptop
3. Touchscreen does not work :(
4. `echo '\\_SB.PCI0.LPCB.EC._Q2A' | sudo tee /proc/acpi/call` (from the `activate-touch-hack.service`)
5. Touchscreen works :)

## Other test:

While looking at powertop I noticed that the touchscreen was using a bit of power. So I was wondering if I could disable it.

I could: ://askubuntu.com/a/1442902

To disable:
- `sudo echo "0003:056A:5144.000E" > /sys/bus/hid/drivers/wacom/unbind`
- `sudo echo "0003:056A:5144.000F" > /sys/bus/hid/drivers/wacom/unbind`

To endable:
- `sudo echo "0003:056A:5144.000E" > /sys/bus/hid/drivers/wacom/bind`
- `sudo echo "0003:056A:5144.000F" > /sys/bus/hid/drivers/wacom/bind`
 
Does not work after sleep, even when I fix the output redirect by replacing it with tee. A bit late to check for alternative solutions. The reason that does not work, is that the touchscreen is not even woken up, so it cannot be bound to the wacom driver.
 
So at least my previous journey was not for nothing.

## Was it worth it?

Like I said, I wanted to keep Thunderbolt, so I could use a dock at my desk. That way I'd have a dedicated device that I could use day to day with Linux, without having to dual boot my main PC. It's also more power efficient.

Speaking of power efficiency. The error does not appear when using the Sleep State "Windows" in the bios (instead of "Linux"). There also is a bug report for the Linux kernel: https://bugzilla.kernel.org/show_bug.cgi?id=203667 where someone claims the Windows mode should be good enough by now. However during my very scientific testing (looking how much battery % is used over time when the laptop is simply closed) I was able to determine that the Linux sleep mode is roughly 4 times as efficient as Windows.

# What next

AFAIK a custom installed kernel module needs to be recompiled after a kernel update.

Apparently DKMS can do that for you.

Reading the documentation of DKMS and `man dkms` (specifically the `add` action) makes it seem like executing `sudo make dkms-add` did already set everything up correctly.

Lets wait for the next kernel update.

Also Fedora uses a different Tool to manage self compiled kernel modules (akmod), I did not feel like looking into that yet.

## The first update

The first update came. The kernel module was compiled, but not installed, even thoudh everything seems to have been configured correctly.

Anyways a `sudo modprobe acpi_call` later it worked again.

## A bit later

The Kernel module is never loaded after reboot. Therefore I created a file `/etc/modules-load.d/70-acpi_call.conf` that adds it after boot.

---

# The destination

So that was a lot of talking. But what does it boil down to?

## Install prerequisites:

`sudo dnf install kernel-devel kernel-headers dkms`

## Install acpi_call

First: check if you have secure boot enabled: `mokutil --sb-state`.
If secure boot is enabled you first need to install DKMS' signing key.
The following command will ask for a password.
This password will only be asked one more time by the BIOS, so make it as easy or secure as you like (it can even be blank):

`sudo mokutil --import /var/lib/dkms/mok.pub`

When rebooting the system you will see a prompt to enroll the MOK. It will look like this: https://github.com/dkms-project/dkms#secure-boot

After rebooting and enrolling the MOK, execute the following commands:

```bash
# with wherever you keep your code projects as your current working directory
git clone https://github.com/nix-community/acpi_call.git
cd acpi_call
sudo make dkms-add
sudo make dkms-build
sudo make dkms-install
sudo make modprobe-install
```

After that you should have a "file" `/proc/acpi/call`.

## Automatically add the module

On my system DKMS successfully recompiles the kernel module after a kernel update.
However the module is not loaded automatically after a reboot.

To fix that create a file `/etc/modules-load.d/70-acpi_call.conf`:

```
# Load acpi_call required for activate-touch-hack
acpi_call
```

The filenames in `/etc/modules-load.d` should start with a 2 digit number from 60-90 according to `man modules-load.d`

## Create service:

Create the file `/etc/systemd/system/activate-touch-hack.service` (would require root permissions so use `sudo vim` or whatever):

```
[Unit]
Description=Touch wake Thinkpad X1 Yoga 3rd gen hack
After=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target

[Service]
ExecStart=/bin/sh -c "echo '\\_SB.PCI0.LPCB.EC._Q2A'  > /proc/acpi/call"

[Install]
WantedBy=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

And enable the service:
```
sudo systemctl daemon-reload
sudo systemctl enable activate-touch-hack.service
```

Now when you close and open your Laptop it should work.
I rarely manage to glitch it out when I maybe closed and opened it too quickly,
but in that case you can run `sudo systemctl start activate-touch-hack.service` or simply close and open it (turn it off and on) again.
During regular operation the workaround has been solid.
