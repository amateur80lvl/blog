# Dissecting Armbian's board support package

Being tired of laziness and drunkenness last evening I decided to give Armvuan a try.
I typed
```
./prepare.sh
./armbian/compile.sh armvuan-build BOARD=myboard BRANCH=current RELEASE=daedalus
```

and went to bed.

When I woke up and rubbed my eyes I found a vomit on the display. Not mine:
```
[🚸] Command failed 3 times, giving up [ chroot_sdcard_apt_get_install armbian-firmware
armbian-bsp-cli-myboard-current linux-dtb-current-rockchip64
linux-image-current-rockchip64 linux-u-boot-myboard-current ]
[💥] Error 1 occurred in main shell [ at /home/mine/projects/armvuan/armbian/
lib/functions/cli/cli-armvuan.sh:470
                  armvuan_blend() --> lib/functions/cli/cli-armvuan.sh:470
                cli_armvuan_run() --> lib/functions/cli/cli-armvuan.sh:105
        armbian_cli_run_command() --> lib/functions/cli/utils-cli.sh:136
                 cli_entrypoint() --> lib/functions/cli/entrypoint.sh:176
                           main() --> ./armbian/compile.sh:50
 ]
[💥] Cleaning up [ please wait for cleanups to finish ]
```

The log had no further explaination of the problem.
Looking at earlier messages I noticed the output directory in `armbian/.tmp/rootfs-UUID`.

Okay, pathologist's best remedy is drink, I had a long swallow and turned
to the smelly stuff on my table:
```
chroot armbian/.tmp/rootfs-UUID
apt install armbian-firmware armbian-bsp-cli-myboard-current \
  linux-dtb-current-rockchip64 linux-image-current-rockchip64 \
  linux-u-boot-myboard-current
```

APT puked in no time:

```
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 armbian-bsp-cli-myboard-current : Depends: base-files (>= 24.5.1)
but 12.4devuan3 is to be installed
                                      Recommends: toilet but it is not
going to be installed
E: Unable to correct problems, you have held broken packages.
```

Hm, well, Armbian guys went above and beyond but Devuan is notoriously stable.
The latter is good, that saved me from the recent xz shit.
So I wouldn't start pursuing latest features, but I have to fix the problem somehow.
Obviously, by getting rid of `armbian-bsp-cli` package.

Let's look inside, given we're still in the chrooted environment:
```
cd /root
apt download armbian-bsp-cli-myboard-current
mkdir armbian-bsp-cli-myboard-current
dpkg-deb -R armbian-bsp-cli-myboard-current*.deb armbian-bsp-cli-myboard-current
apt install tree
tree  armbian-bsp-cli-myboard-current
```

From what I observe, I find the following looks absolutely unnecessary for Armvuan:
* `etc/armbian*` files
* `etc/cron*` directories
* `etc/network`
* `etc/NetworkManager`
* `etc/profile.d`
* `etc/systemd`
* `etc/update-mot.d`
* `etc/X11`
* `lib`
* `usr/sbin`
* `usr/share`
* `usr/var`

The following is worth keeping:
* `/etc/apt`
* `/etc/initramfs`
* `/etc/kernel`
* `/etc/modprobe.d`
* `/etc/sysfs.d`
* `/etc/udev`

However, I don't see any board-specific stuff so far.

Required files:
* `etc/armbian-release`: needed by `etc/initramfs/post-update.d/99-uboot` at least

`usr/bin` contains `armbianmonitor` only which looks pretty good tool,
although it's just a bash script that pretty prints the data from `/proc/` and `/sys`.

Finally, the last and the most interesting directory is `usr/lib`.
It contains Armbian utilities. However, I still see no board-specific stuff there.
Even `armbian-hardware-optimization` is a bash script that embeds tweaks for all boards
and why the deb package contains myboard in its name is still a mistery.

From Armvuan README:

> Armvuan is shipped with the following services enabled:
> * armbian-hardware-optimization
> * armbian-ramlog

It seems I need only a few scripts, but I don't want to dig out dependencies so
the easy way is to keep the entire `usr/lib`.

The only question I have at this point is why the deb package needs such a high version
of base files?

Obviously, some scripts may need some specific features or it's just a maintainer's quirk.

Only empirical way can give an answer.

The best way could be repacking the deb package, but that leads to a PPA, maintenance,
and other headaches. As a quick hack (all software is written this way, isn't it?)
I'll just amend `cli-armvuan.sh` to pull all the necessary files.
Not a good way, not easy to upgrade, but I have no better idea in terms of simplicity.

## Dependencies

The list of dependencies should be updated with:
* linux-base
* parallel

## Problems

They use `/usr/bin/bash` for `initrd_files_to_hash`. Usrmerge was a long thread on
Devuan mailing list. I don't remember their final desicion, but daedalus keeps
bash in `/bin`.

Not sure how much this is important, the build script complains, but continues
to run normally.

## Results

Flashed microSD card, corrected IP address, hostname, added SSH key.
Inserted into my board and powered it on.
Successfully connected by SSH.

It works. That's impressive! No fucking systemd! Devuan is great.
Armbian does use fucking systemd, however, their board support is amazing.
They are great too!

But wait, let's give `armbianmonitor` a try. It needs `bc` but it does not show
CPU temperature even if I install that missing `bc`!
After glancing at the script I found it reads temperature from
`/etc/armbianmonitor/datasources/soctemp` which is a symbolic link that depends
on board type.

This link is created by `prepare_temp_monitoring` function in
`/usr/lib/armbian/armbian-hardware-monitor`.
I don't want this service but I'd like to call this function at startup.
Let it be a part of `/etc/init.d/armbian-hardware-optimization`:
```bash
# tweaks for armbianmonitor:
. /usr/bin/armbian/armbian-hardware-monitor
prepare_temp_monitoring
```

Made changes to [armvuan](https://github.com/amateur80lvl/armvuan/commit/2698d5a97001e524bcf7476aae15714b7baa3447).
Enjoy, none of its users!
