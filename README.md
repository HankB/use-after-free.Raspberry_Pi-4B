# use-after-free.Raspberry_Pi-4B

notes on collecting information on this issue

* Pi 4B with 8GB RAM
* Booting from an SSD

## procedure

* install image `xzcat 20220121_raspi_4_bookworm.img.xz >/dev/sdf` on other desktop
* boot
* `adduser hbarta` to facilitate remote login
* `hostnamectl set-hostname up`
* reboot to complete host name change
* Remote in, `su` to root and
    * `apt-mark hold raspi-firmware linux-image-arm64`
    * `apt update`
    * `apt -y upgrade`
    * reboot
    * `apt install task-gnome-desktop`
    * reboot

## Steps to trigger issue

* remote login, su to root and `dmesg --follow`
* login to desktop and open browser
* Mouse to upper left corner to bring up overview
* Grab browser and try to move to next right desktop
* underflow is reported in `dmesg` output.


## Results

First try, on second boot, system mounted root RO (after a slow boot) `<trl><alt><del>` reboot and still going slow - switching to another SSD.

Second try going better - boot takes  ~30s.

### kernel 5.15, desktop installed

```text
Linux up 5.15.0-2-arm64 #1 SMP Debian 5.15.5-2 (2021-12-18) aarch64 GNU/Linux
```

* start `dmesg --follow` via remote before logging into desktop
* log into desktop
* open Firefox
* hit upper left hoit corner to get overview
* Grab Firefox top bar and slide to next right desktop
* DRM error appears on `dmesg` output `drm:drm_crtc_commit_wait [drm]] *ERROR* flip_done timed out`
* Not long after the `underflow; use-after-free.` diagnoistic appears
* Gnome Desktop is now locked up (probably following) the DRM error
* Capture dmesg output, save to dmesg01 <https://paste.debian.net/1236160/>
* Screen now dark and does not respond to mouse or keyboard. `ctrl-alt-Fn` does not bring up text console.
* `ctrl-C` to exit `dmesg` and `shutdown -r now` (from remote session)
* Shutdown and reboot appear normal.
* repeat procedure except wait for system to settle before moving Firefox to other desktop.
* start `System Monitor` and see that CPU has settled to about 10%.
* launch Firefox and wait for CPU to settle to ~10%
* Switch to overview, wait for CPU to settle and slide Firefox over.
* Same result. (My theory that doing this during hiugh CPU load triggered the DRM error seems not to be the case.)
* `dmesg` output not collected as it seemed to duplicate the previous test.

### kernel 5.16, desktop installed

Followed same procedure as above except after logging in remote via ssh, and starting `dmesg --follow` I waited for 5 minutes to see if the issue and my efforts to trigger it were coincidental.

```text
Linux up 5.16.0-5-arm64 #1 SMP Debian 5.16.14-1 (2022-03-15) aarch64 GNU/Linux
```

* Remote in and run `dmesg --follow`
* delay 5 minutes
* login to desktop
* launch `System Monitor`
* launch Firefox - Firefox reports "unable to restore last session."
* delay 5 minutes
* unlock desktop and go to overview
* Delay 1 minute
* Slide Firefox to next right desktop
* underflow message appears but w/out DRM errors. Desktop does not lock up.
* Capture `dmesg` output dmesg02 <https://paste.debian.net/1236162/>
* Reboot and repeat but without delays. Same resul;t (no DRM error but still get `underflow; use-after-free.`)
* `dmesg` output similar to previous and not saved.

### kernel 5.17 from experimental

```text
Linux up 5.17.0-rc8-arm64 #1 SMP Debian 5.17~rc8-1~exp1 (2022-03-14) aarch64 GNU/Linux
```

Add experimental repo per instructions at <https://wiki.debian.org/HowToUpgradeKernel>
(NB: Cannot copy instructions from page because leading space causes here document to not work with `bash`.)

```text
root@up:~# cat /etc/apt/preferences.d/linux-kernel
Package: *
Pin: release o=Debian,a=experimental
Pin-Priority: 102
root@up:~# apt policy
Package files:
 100 /var/lib/dpkg/status
     release a=now
 500 http://deb.debian.org/debian bookworm/non-free arm64 Packages
     release o=Debian,a=testing,n=bookworm,l=Debian,c=non-free,b=arm64
     origin deb.debian.org
 500 http://deb.debian.org/debian bookworm/contrib arm64 Packages
     release o=Debian,a=testing,n=bookworm,l=Debian,c=contrib,b=arm64
     origin deb.debian.org
 500 http://deb.debian.org/debian bookworm/main arm64 Packages
     release o=Debian,a=testing,n=bookworm,l=Debian,c=main,b=arm64
     origin deb.debian.org
Pinned packages:
root@up:~# echo "deb http://deb.debian.org/debian experimental main" >> /etc/apt/sources.list
root@up:~# apt update
Hit:1 http://security.debian.org/debian-security bookworm-security InRelease         
Hit:2 http://deb.debian.org/debian bookworm InRelease                                
Get:3 http://deb.debian.org/debian experimental InRelease [75.4 kB]
Get:4 http://deb.debian.org/debian experimental/main arm64 Packages [373 kB]
Get:5 http://deb.debian.org/debian experimental/main Translation-en [233 kB]
Fetched 682 kB in 2s (303 kB/s)                               
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
1 package can be upgraded. Run 'apt list --upgradable' to see it.
root@up:~# apt install -t experimental linux-image-5.17.0-rc8-arm64-unsigned
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Suggested packages:
  linux-doc-5.17 debian-kernel-handbook
The following NEW packages will be installed:
  linux-image-5.17.0-rc8-arm64-unsigned
0 upgraded, 1 newly installed, 0 to remove and 53 not upgraded.
Need to get 57.3 MB of archives.
After this operation, 403 MB of additional disk space will be used.
Get:1 http://deb.debian.org/debian experimental/main arm64 linux-image-5.17.0-rc8-arm64-unsigned arm64 5.17~rc8-1~exp1 [57.3 MB]
Fetched 57.3 MB in 4s (13.6 MB/s)                                
Selecting previously unselected package linux-image-5.17.0-rc8-arm64-unsigned.
(Reading database ... 148953 files and directories currently installed.)
Preparing to unpack .../linux-image-5.17.0-rc8-arm64-unsigned_5.17~rc8-1~exp1_arm64.deb ...
Unpacking linux-image-5.17.0-rc8-arm64-unsigned (5.17~rc8-1~exp1) ...
Setting up linux-image-5.17.0-rc8-arm64-unsigned (5.17~rc8-1~exp1) ...
I: /vmlinuz.old is now a symlink to boot/vmlinuz-5.16.0-5-arm64
I: /initrd.img.old is now a symlink to boot/initrd.img-5.16.0-5-arm64
I: /vmlinuz is now a symlink to boot/vmlinuz-5.17.0-rc8-arm64
I: /initrd.img is now a symlink to boot/initrd.img-5.17.0-rc8-arm64
/etc/kernel/postinst.d/initramfs-tools:
update-initramfs: Generating /boot/initrd.img-5.17.0-rc8-arm64
root@up:~# 
```

```text
Linux up 5.17.0-rc8-arm64 #1 SMP Debian 5.17~rc8-1~exp1 (2022-03-14) aarch64 GNU/Linux
```

* remote in and start `dmesg --follow`
* Log in to desktop, open Firefox and try to move to next reight desktop (as before.)
* `underflow; use-after-free.` appears not without DRM errors seen with earlier kernels.
* Desktop is not locked up - still usable.
* Capture `dmesg` output dmesg03 <https://paste.debian.net/1236164/>


## incidental 

Tested WiFi/Ethernet for someone else - dump of `daemon.log` at <https://paste.debian.net/1236170/>