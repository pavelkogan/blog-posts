---
layout: post
title: Linux Mint encryption
---

In my [previous post](/2014/05/23/luks-full-disk-encryption/) on full disk encryption I described how to avoid having to enter your passphrase twice. That method, however, only works on Arch. Here's how to do it on Linux Mint[^mint].

The initial setup process (with LVM, LUKS and GRUB) is the same as on Arch, but instead of editing `/etc/mkinitcpio.conf`, which doesn't exist on Mint, create `/etc/crypttab`:

    lvm /dev/sda1 none luks

So far, so good. The `crypttab` man page even talks about how you can point to a keyfile in the third column ("none" above). Unfortunately, if you do so, it doesn't actually work. If you read the `cryptroot` script[^script], you find that the keyfile is only ever used as an argument to a keyscript.

Oddly enough, although there are several provided scripts, each doing various exotic things, none of them seem to handle the simple case of using the key to just decrypt the drive.

What these scripts have in common is that they take the "keyfile" as an argument, and their output is used as the key. So, all we need to do is provide a "script" that will take a filename and output the file's contents:

    lvm /dev/sda1 /crypto_keyfile.bin luks,keyscript=/bin/cat

Now, the `cryptroot` hook[^hook] will copy the `cat` executable into the ramdisk, and during boot `cat` will send the keyfile's contents to `cryptsetup`.

All that's left is to ensure the keyfile is available before the drive is decrypted by copying it into the ramdisk too. There's no convenient `FILES` option like in Arch, so you'll have to make a custom hook. Luckily, it's trivial:

    #!/bin/sh
    cp /crypto_keyfile.bin "${DESTDIR}"

Put it in `/etc/initramfs-tools/hooks/` and make it executable:

    chmod +x /etc/initramfs-tools/hooks/crypto_keyfile

Recreate the ramdisk:

    update-initramfs -u

Check that everything is where it should be with `lsinitramfs` and reboot.

---

[^mint]: It should also work with Debian/Ubuntu (or any distro that uses `initramfs-tools`)
[^script]: in `/usr/share/initramfs-tools/scripts/local-top/`
[^hook]: in `/usr/share/initramfs-tools/hooks/`
