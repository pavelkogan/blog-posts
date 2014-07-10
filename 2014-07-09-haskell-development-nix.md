---
layout: post
title: Haskell development with Nix
---

Haskell development with cabal was greatly improved by the addition of cabal sandboxes, but cabal hell and issues with non-Haskell dependencies persist. For example, building the gtk bindings is impossible with the latest version of cabal; I was forced to work around this by installing them with the system package manager. When the necessary packages at the right version are unavailable in the package manager though, an alternative solution needs to be found. Enter [Nix](https://nixos.org/nix/).

I actually first encountered Nix a few years ago, trying out [NixOS](https://nixos.org/) for a while before returning to better supported distributions. Even now, the downsides seem to outweigh the advantages, at least for a single-user system, but with recent attention to using Nix for Haskell development, I decided to try using it solely for that, without dedicating my entire system to NixOS.

## nix-shell

The major draw of developing with Nix over cabal alone is being able to easily switch out which dependencies are used, without having to delete, rebuild, wait, worry about whether the builds will succeed with the new versions, and so on. The way this is done is by having `nix-shell` set up an environment where all the dependencies are available. Fortunately, enumerating the dependencies doesn't have to be done by hand; the `cabal2nix` tool can be used to create Nix expressions from your project's cabal file:

    cabal2nix projectname.cabal --sha256=0 > shell.nix

The dummy sha256 hash is needed to prevent `cabal2nix` from trying to download the package from Hackage.

`cabal2nix` generated Nix expressions are intended to be added to [nixpkgs](https://nixos.org/nixpkgs/), so a few edits are needed to have it work automatically with `nix-shell`:

1. Replace `sha256 = "0";` with `src = "./.";`
2. Replace the list of function parameters (`{ cabal, mtl, ... }:`) with

       { haskellPackages ? (import nixpkgs {}).haskellPackages }:

   If any of the parameters are not Haskell packages, keep them in the list, but add a default value, as above.
3. Prepend `with haskellPackages;` to the expression body, i.e.

       with haskellPackages; cabal.mkDerivation (self: {
       ...

4. Optionally, if you don't have cabal installed globally, add `cabalInstall` to `buildTools`. So, if no other build tools are used:

       buildTools = [ cabalInstall ];

> A potential issue can arise if the name of a non-Haskell package you need conflicts with the name of a Haskell package. In that case, you need to move the `with haskellPackages;` inside the expression body, like so:
>
>     haskellPackages.cabal.mkDerivation (self: {
>       ...
>       buildTools = with haskellPackages; [ cabalInstall ];
>       buildDepends = with haskellPackages; [ ... ];
>       ...
>
> Alternatively, add the package to `extraLibraries`:
>
>     extraLibraries = with import <nixpkgs> {}; [ pkgname ];
>

Now, calling `nix-shell` in the project directory will drop you into a shell with all the necessary packages in your `$PATH`, or you can hook up something like `nix-shell --command='cabal build'` to your build system.

## Missing dependencies

Not all packages on Hackage are in nixpkgs. If your project depends on one of those, it's possible to add it yourself. There are two ways to do this; you can clone the [nixpkgs repository](https://github.com/NixOS/nixpkgs) and modify it[^1], or you can just add it locally. I will describe the latter.

To add Nix expressions to the local nixpkgs collection, you need to create a `~/.nixpkgs/config.nix` file. Now, the Nix wiki gives an outline of what this file should look like, but with the Nix documentation being as it is[^2], most of the functions and attributes used are undocumented, so I wasn't able to tell how it's supposed to work. Not being comfortable with copy-pasting an opaque block of code, I wrote a clearer version, which probably works differently, but does the job nonetheless:

~~~
{
  packageOverrides = pkgs: rec {
    haskellPackages = with pkgs.haskellPackages; pkgs.haskellPackages // rec {
      packageName = callPackage ./haskell/package-name.nix {};
      ...
    };
  };
}
~~~

With this, packages can be added much the same way they would be to the main nixpkgs collection:

- Add the `callPackage` line to `config.nix` for the package, as above.
- Use `cabal2nix` to generate the Nix expression:

      cabal2nix cabal://package-name > ~/.nixpkgs/haskell/package-name.nix

Simple.

## Local package dependencies

If you want to have your project depend on a local package not on Hackage, that's just as easy; all it takes is replacing the `sha256` attribute with a `src` attribute pointing to the directory containing the package (as with `shell.nix` for `nix-shell` project support).

## Conclusion

Using Nix expressions to set up your development environment can provide a lot of flexibility, allowing you to do things like switch between libraries with or without profiling, or even between compiler versions, which has been described elsewhere[^3]. With this post I've shown, I hope, how a basic Nix setup can be created with very little up-front effort, streamlining workflow and providing a base to build on should that flexibility become needed.

---

[^1]: <https://nixos.org/wiki/Contributing_to_nixpkgs>
[^2]: That is to say, not very good.
[^3]: My use of Nix for Haskell development is heavily inspired by "[How I Develop with Nix](http://ocharles.org.uk/blog/posts/2014-02-04-how-i-develop-with-nixos.html)" and "[My experience with NixOS](http://fuuzetsu.co.uk/blog/posts/2014-06-28-My-experience-with-NixOS.html)".
