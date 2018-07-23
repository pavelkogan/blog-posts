---
layout: post
redirect_from: /2014/07/13/nix-shell-on-linux-mint/
title: nix-shell on Linux Mint
---

After spending hours tearing out my hair trying to figure out why `nix-shell` wasn't working on Linux Mint, which included digging through the Perl source code `nix-shell` is written in (I **hate** Perl), I find that it's caused by a call to `mint-fortune` in `/etc/bash.bashrc`.

Fixed by commenting out one line. Damn it.
