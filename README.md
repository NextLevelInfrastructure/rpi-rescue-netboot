# rpi-rescue-netboot

Some people netboot their Raspberry Pi and then mount an NFS root
or an iSCSI root. Other people netboot their Raspberry Pi in order to
remotely control the kernel and/or device configuration but then mount
a root partition on an SD/eMMC or USB Mass Storage Device.

If you are some people or other people, this project is not for you.

rpi-rescue-netboot is for users of Raspberry Pi 4B or Compute Module 4
who want to boot from the network to:

* run a performant, maintainable, configuration controlled version of
  Raspberry Pi OS on multiple machines, which may have no local stable
  filesystem or which may have intermittent, low-bandwidth,
  high-latency, or metered network connectivity;

* interactively repair a corrupted filesystem on a remote or headless
  machine; or

* interactively provide the passphrase to an encrypted filesystem on a
  remote or headless machine.

We also hope that our project documentation can be useful to people
who want to learn more about the Raspberry Pi boot process.

## What rpi-rescue-netboot is

rpi-rescue-netboot is a unitary Raspberry Pi OS image that needs no
installation or external storage beyond the onboard RAM. Accessing
nothing outside the image itself, rpi-rescue-netboot can accept logins
via ssh or the local serial console. It can run fsck on most common
filesystems. It can use scp and rsync to copy data out of the system
and can also use wget to bring data in.

rpi-rescue-netboot can optionally load a squashfs image from an
external webserver. The (read-only) squashfs is mounted as a lower
layer in overlayfs, with a writeable in-memory tmpfs mounted as an
upper layer. The squashfs image can be a full, unabridged installation
of Raspberry Pi OS or any other OS, limited only by the size of
available RAM that can be dedicated to squashfs. rpi-rescue-netboot
execs systemd or another suitable init process from the squashfs image
to continue multi-user boot.

rpi-rescue-netboot is a set of Ansible playbooks, along with
documentation, that hopefully makes it easier to build, customize, and
deploy remote and/or headless Raspberry Pi 4B and CM4 devices that are
capable of performing network boot.

rpi-rescue-netboot consists almost entirely of object and executable
code copied directly from Raspberry Pi OS without even recompiling
it. This framework allows you to customize the system significantly
without requiring cross-compilation support. The cost of this is that
rpi-rescue-netboot is not as small as it could be if it used, for
example, buildroot.

rpi-rescue-netboot may be loaded using any boot mode supported by the
Raspberry Pi 4B or Compute Module 4 (CM4), but it is primarily
intended to be loaded by HTTP boot mode or NETWORK (tftp) boot mode.

## Contents of this repository

This README provides an overview



## FAQs

* **Should I use HTTP boot or NETWORK (tftp) boot for my Raspberry Pi
    4B or CM4?** Both work. If you have a functioning tftp boot
    service that you're happy with, stick with it. If you're just
    starting, be aware that tftp boot has a lot of corner cases and
    poorly documented requirements. HTTP boot is more likely to "just
    work." And HTTP boot will be much more efficient for data transfer
    outside the very local network.

* **Can I use this to PXE boot?** Many people seem to use "PXE boot"
    as a synonym for "network boot," and if that's what you mean, yes
    rpi-rescue-netboot can do it. But the Raspberry Pi and Compute
    Modules have never been able to do an actual PXE boot. True PXE
    machines can slot into large-scale PXE architecture for things
    like zero-touch provisioning, and Raspberry Pi does not do that,
    and rpi-rescue-netboot doesn't do it either.

* **Is this a replacement for iPXE?** rpi-rescue-netboot is larger
    than iPXE and has more capabilities than iPXE does. iPXE can be
    scripted and can provide a very good approximation of "true PXE"
    by chainloading, so if that's important to you you should
    use iPXE and not rpi-rescue-netboot.

* **Why did you build this?** We run quite a number of remote,
    headless devices, many of which are difficult to physically
    visit. So we need ways to investigate the operation of our systems
    when they fail. Network boot of a complete system enables us to

* **No, but why couldn't you use something pre-existing?** System
    Rescue doesn't support ARM. Buildroot makes highly configurable
    images that are about as small as can be given the modules
    included in the config, but it requires real configuration effort
    and starts to pull you away from stock Raspberry Pi OS (or even
    away from the custom kernel you may run in production).
