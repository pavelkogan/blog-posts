---
title: Full disk encryption with LUKS (including /boot)
redirect_from: /2014/05/23/luks-full-disk-encryption/
layout: post
---

> Update (25/01/15): I wrote a [new post]({{ site.baseurl }}{% post_url 2015-01-25-linux-mint-encryption %}) about how to achieve the same thing with Linux Mint.

While looking for information about how to encrypt my laptop's hard drive, among the repeated claims that the partition on which `/boot` resides must remain unencrypted, I found the suggestion that GRUB should be able to handle cryptography since it can be set up with a hashed password.

Being too lazy to want to deal with a separate boot partition, I went looking to see what modules GRUB can load, and there they were[^1]: `crypto.mod`, `cryptodisk.mod` and even `luks.mod`!

Since there don't appear to be any instructions on how to fully encrypt a system including `/boot`, I've decided to make a short guide on how to do it.

> This guide will describe setting up an encrypted Arch Linux system. The procedure is mostly distribution-agnostic, but note that anything involving `mkinitcpio` is Arch Linux specific and must be replaced if another distribution is used.

## Set up partitions (LVM on LUKS)

This is well-documented elsewhere, so I won't be explaining it. If something is unfamiliar to you, I recommend reading the [ArchWiki page](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS) on the subject before proceeding.

~~~
parted -s /dev/sda mklabel msdos
parted -s /dev/sda mkpart primary 2048s 100%
cryptsetup luksFormat /dev/sda1
cryptsetup luksOpen /dev/sda1 lvm
pvcreate /dev/mapper/lvm
vgcreate vg /dev/mapper/lvm
lvcreate -L 4G vg -n swap
lvcreate -L 15G vg -n root
lvcreate -l +100%FREE vg -n home
mkswap -L swap /dev/mapper/vg-swap
mkfs.ext4 /dev/mapper/vg-root
mkfs.ext4 /dev/mapper/vg-home
mount /dev/mapper/vg-root /mnt
mkdir /mnt/home
mount /dev/mapper/vg-home /mnt/home
~~~

## Install Linux

At this point you should be in a live system with all partitions mounted, so you can go ahead and run the install. Just be sure not to reboot once it's done.

> Don't forget to add the `lvm2` and `encrypt` hooks to `/etc/mkinitcpio.conf` and run
>
>     mkinitcpio -p linux
>

## Configure GRUB

With `/boot` on an encrypted device, `grub-mkconfig` should have GRUB load the necessary modules to decrypt and mount it[^2].

`grub-install`, on the other hand, will refuse to work, complain about /boot being encrypted, and demand that `GRUB_ENABLE_CRYPTODISK=1` be added to the config. This is a [bug](https://savannah.gnu.org/bugs/?41524) (in the error message). Instead, add

    GRUB_ENABLE_CRYPTODISK=y

to `/etc/default/grub`.
Now, before trying to find and load the initial ramdisk, GRUB will ask for a passphrase to decrypt `/dev/sda1`.

Finally, add the `cryptdevice` kernel parameter[^3]

    GRUB_CMDLINE_LINUX="cryptdevice=/dev/sda1:lvm"

and run

    grub-mkconfig -o /boot/grub/grub.cfg
    grub-install /dev/sda

Reboot, and that's it. You now have a fully encrypted system.

## Bonus: Login once

You've probably noticed that there remains the minor annoyance of having to decrypt your drive twice: once for GRUB and once for the kernel. Evidently, when GRUB passes control to the kernel, the encrypted drive is dismounted.

There is, however, a way to open a LUKS device without entering a passphrase: with a keyfile. The `encrypt` hook can take the file specified in the `cryptkey` kernel parameter (default: `/crypto_keyfile.bin`) and use it to unlock the `cryptdevice`.

~~~
dd bs=512 count=4 if=/dev/urandom of=/crypto_keyfile.bin
cryptsetup luksAddKey /dev/sda1 /crypto_keyfile.bin
~~~

I tried various methods to get GRUB to load the keyfile into memory and pass it to the kernel, without success. Then, I realised that the `initrd` image is itself something GRUB loads into memory, and `mkinitcpio.conf` has a very convenient `FILES` option...

    FILES=/crypto_keyfile.bin

Run `mkinitcpio` again, and when you reboot, you'll only need to enter your password once.

### Security considerations

While the computer is off, the keyfile is stored inside the encrypted drive, so it is secure. When the computer is on, however, the keyfile is unencrypted, with a copy on the ramdisk. So, you should probably make sure only _root_ can access these files:

~~~
chmod 000 /crypto_keyfile.bin  # actually, even root doesn't need to access this
chmod -R g-rwx,o-rwx /boot     # just to be safe
~~~

The keyfile will probably also be retained in memory, but so is the LUKS master key. If the attacker has enough access to your system for that to be a problem, encryption is moot. So long as there are no copies of the keyfile anywhere else, you should be fine. In other words, don't back it up or reuse it elsewhere.

---

[^1]: in `/boot/grub/i386-pc/`

[^2]: If it doesn't---check `/boot/grub/grub.cfg`---then you can add `cryptodisk` and `luks` to `GRUB_PRELOAD_MODULES`.

[^3]: <https://wiki.archlinux.org/index.php/GRUB#Root_encryption>
