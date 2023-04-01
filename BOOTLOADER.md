# Raspberry Pi and CM4 Bootstrap Process

The execution flow of control during [rPi4 bootstrap](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#raspberry-pi-4-boot-flow)
goes like this:

  * first-stage bootloader from ROM
  * second-stage bootloader from EEPROM
  * firmware `/start4.elf` with linker info `/fixup4.dat`
  * kernel `/kernel8.img` [for 64-bit kernels]

Even the first stage bootloader expects the first partition of the
boot image to be a FAT32 partition called the [boot
filesystem](https://www.raspberrypi.com/documentation/computers/configuration.html#the-boot-folder). This
filesystem is mounted rw on /boot during normal operation. All
absolute paths in this document are paths within this filesystem.

1. The first stage bootloader is read from the BCM2711 SoC ROM.

1a. If the device is a CM4 carrier board and the BOOT switch is on,
go to step 1d. ("BOOT switch on" means nRPIBOOT is being held active low.)

1b. If OTP has not disabled self-update and if a valid `/recovery.bin`
is on the SD/eMMC, run it to update the SPI EEPROM.

1c. If SPI EEPROM has a valid second stage bootloader, run it.

1d. Retry forever until `/recovery.bin` can be run from USB *device*
boot, or until an `rpiboot` host commands via USB a switch to USB-MSD
mode. Note that the ROM has no provision for running `/recovery.bin`
from USB-MSD boot (but see step 2c).

2. The second stage bootloader image is read from the SPI EEPROM.

2a. The second stage image is a combination of a bootloader executable
and EEPROM configuration data. To replace the EEPROM configuration
data for an image with new data, use `rpi-eeprom-config`; see
below. `rpi-eeprom-config` is often run with `--config /boot/boot.cfg`
but **the contents of `/boot.cfg` are ignored by the boot flow** and
only the configuration data encoded into the image is used.

2b. The second stage bootloader loops through the boot modes defined
in the BOOT_ORDER parameter in EEPROM configuration data and attempts
to load firmware via each mode in turn. 

2c. If ENABLE_SELF_UPDATE=1 in the EEPROM configuration data *and* the
boot mode is NETWORK (tftp) or USB-MSD *and* `/pieeprom.upd` and
`/pieeprom.sig` are valid *and* the current EEPROM image differs from
the `/pieeprom.upd` image, then the update image is written to the SPI
EEPROM and the system is reset.

3. The firmware image is read from `/start4.elf` with linker fixups
defined in `/fixup4.dat`.

3a. The firmware reads `/config.txt`. If `start_file` and `fixup_file`
are specified, those are loaded as a replacement firmware image and
execution continues there.

3b. The firmware loads the device tree, loads any device tree overlays
`dtoverlay` and sets any device tree parameters `dtparam` specified in
`/config.txt`, and configures the overlaid, parameterized device
tree. Note that any Compute Module requires a dtoverlay to enable any
UART, though if a UART console is enabled in the bootloader no
dtoverlay is required to enable uart1.

3c. The firmware loads the kernel image. If a `kernel` option is
defined, that kernel is loaded; otherwise if `arm_64bit` is nonzero,
`/kernel8.img` is loaded; otherwise `/kernel7l.img` is loaded.

3d. The firmware uncompresses the kernel image if it is
compressed. Only the magic number is used to determine compression
status.  Both compressed and uncompressed kernels use a simple .img
extension.

3e. If an `initramfs` option is defined (or both `ramfsfile` and
`ramfsaddr` are defined), the specified file (a gzipped cpio archive)
is loaded. See update-initramfs(8).

3f. `/cmdline.txt` (as modified/augmented by the device tree) is
passed as the kernel command line. Note that `/cmdline.txt` should not
end with an EOL character. When initramfs is the root filesystem, this
should contain `root=/dev/ram rootfstype=auto rdinit=/sbin/init` .

3g. If `/ssh` or `/ssh.txt` is present (their contents are ignored), sshd
is started at boot time. (How and when this happens is unclear to Daniel.)

4. The kernel unpacks the initramfs, mounts it as the initial root
filesystem, and spawns an instance of `init` read from the path
specified in`rdinit=`. Any device drivers and executables needed to
mount any other filesystems or otherwise configure the system to a
useful state must be present in the initramfs.

## What We Are Doing

The first partition (FAT32) of the boot image is the boot
filesystem. It contains /config.txt, /start4.elf, /fixup4.dat, the
device tree files, /kernel8.img, /cmdline.txt, and
/initramfs.cpio.gz. Everything is stock Raspberry Pi OS except for
/config.txt, /cmdline.txt, and /initramfs.cpio.gz

/initramfs.cpio.gz will contain /sbin, /bin, /lib, and a minimal /usr
and /etc. It will be sufficient to load the kernel modules for
squashfs and overlayfs, and run mount, wget, dropbear, and a shell.

(We could set cmdline.txt to set root on SD/eMMC for normal boot, but
we'd have to make the UUID of the partition the same on all devices if
we want a single image for all devices. We won't do this because it
doesn't seem useful except for testing.)

Initramfs /sbin/init will try to wget the squashfs image. If wget
succeeds, it will mount the squashfs on /usr, mount overlayfs, and exec
systemd for a normal boot. If wget fails, it will start dropbear and
exec a console shell.

Raspberry Pi OS ships with /bin as a symlink to usr/bin, /lib as a
symlink to usr/lib, /sbin as a symlink to usr/sbin, /dev as a
mountpoint for devtmpfs, /sys as a mountpoint for sysfs, /proc as a
mountpoint for proc, /boot as a mountpoint for the FAT32 boot
filesystem, /run as a mountpoint for tmpfs. /mnt, /media, /opt, and
/srv are empty. /root and /home are not needed. /etc is 4.9
MB. /var/lib is 510 MB, almost all of which is docker and apt.

Thus, for directories on the initrd:

* /etc is copied entire from the running installed system.
* /sbin, /lib, /bin are small subsets of the running installed system.
* /var is created as a tree with mostly empty subdirectories and a few
  symlinks.
* /usr is created with symlinks to /sbin, /lib, and /bin.
* /dev, /sys, /proc, /run, /mnt, /media, /opt, /srv, /root, and /home
  are created empty.



## Notes

Regarding step 3f: a `/cmdline.txt` containing:

```
console=serial0,115200 console=tty1 root=PARTUUID=52a352bd-02 rootfstype=ext4 fsck.repair=yes rootwait
```

is modified by the device tree to produce `/proc/cmdline` of:

```
coherent_pool=1M 8250.nr_uarts=1 snd_bcm2835.enable_headphones=0 snd_bcm2835.enable_headphones=1 snd_bcm2835.enable_hdmi=1 snd_bcm2835.enable_hdmi=0  smsc95xx.macaddr=E4:5F:01:3F:BD:6F vc_mem.mem_base=0x3ec00000 vc_mem.mem_size=0x40000000  console=ttyS0,115200 console=tty1 root=PARTUUID=52a352bd-02 rootfstype=ext4 fsck.repair=yes rootwait
```

## Stuff

To set up rpiboot:

```
sudo apt install libusb-1.0-0-dev python3-pycryptodome
git clone --depth=1 https://github.com/raspberrypi/usbboot
cd usbboot; make
mkdir -p nln; cd nln

# Generate a 2048-bit RSA key (required by bootloader)

# openssl genrsa -out rpi-httpboot-private.pem 2048
openssl rsa -in rpi-httpboot-private.pem -outform PEM -pubout -out rpi-httpboot-public.pem

# Copy configs/ooboob/rpi/boot.conf to usbboot/nln/bot.com

#latestfw=`rpi-eeprom-update -l`
latestfw=../recovery/pieeprom.original.bin
../recovery/rpi-eeprom-config --config ../nln/boot.conf -p ../nln/rpi-httpboot-public.pem --out pieeprom.nln "$latestfw"

# Generate signature for your eeprom
rpi-eeprom-digest -i pieeprom.nln -o pieeprom.sig

# this will install on a standalone Raspberry Pi
rpi-eeprom-update -d -f pieeprom.nln

# to install on a CM4:
cp pieeprom.nln ../recovery/pieeprom.bin
cp pieeprom.sig ../recovery/
cd ../recovery
../rpiboot -d .

# nlnboot.img is a fat32 partition with kernel and initrd

# Sign the network install image with your PRIVATE key
# Put boot.img and boot.sig on your web server
rpi-eeprom-digest -i nlnboot.img -o nlnboot.sig -k rpi-httpboot-private.pem
```

## Other links

* https://blockdev.io/read-only-rpi/ uses overlayfs to do RAM over NFSv4

* OpenWRT NFS server package: https://openwrt.org/docs/guide-user/services/nas/nfs.server

* https://www.jeffgeerling.com/blog/2022/how-update-raspberry-pi-compute-module-4-bootloader-eeprom

* https://wiki.aalto.fi/download/attachments/84751520/RootFilesystem_for_RPi.pdf?version=1&modificationDate=1386894761177&api=v2

* buildroot

* create squashfs: https://www.berryterminal.com/doku.php/berryboot/adding_custom_distributions

*** OOB

log in as root@
cd /tmp
wget https://downloads.openwrt.org/releases/22.03.3/targets/ath79/generic/openwrt-22.03.3-ath79-generic-glinet_gl-x750-squashfs-sysupgrade.bin
sysupgrade -v /tmp/openwrt-22.03.3-ath79-generic-glinet_gl-x750-squashfs-sysupgrade.bin
