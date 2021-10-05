---
title: Automatically cleaning pacman and yay cache in Arch Linux
date: 2021-03-29 19:16:00 +0200
categories: [Linux, System administration]
tags: [arch linux, pacman, yay, bash, cache, script]
---

Let's talk about something interesting for Arch Linux(-based) distros: automating package cache cleaning.

## Cleaning pacman cache

pacman is the official package manager for Arch. When using it to install a package from official repositories, it gets donwloaded, stored in our system (specifically, in `/var/cache/pacman/pkg`), and finally it is installed and the package file stays there, in case we need to reinstall it in the future (for example, if we have uninstalled it and then we want to install it again), or in case we install a newer version, it doesn't work as expected and we want to downgrade.

However, the package cache is never cleaned automatically and, as we install and upgrade packages, it grows up in size. We could wait until our `/` is full and then cleaning it... or cleaning it automatically, keeping the files that can be useful.

pacman features an operation for cleaning cache of uninstalled packages, `pacman -Sc`, and for cleaning cache for every package, `pacman -Scc`. However, it could be interesting to clean it keeping the two latest versions of each package (the installed one and the previous one), so we always have a previous version for every package, but not every of them, in case we have to downgrade. We can achieve this using `paccache`, a script from `pacman-contrib` package.

Running `paccache -rk2`, we clean the package cache keeping just the two latest versions of each package. Furthermore, we can use `-u` option for cleaning just the cache of uninstalled packages so if, for example, we want to keep the two latest versions for installed packages and just the latest for uninstalled packages (in case we want to reinstall them in the future, for example for make dependencies of AUR packages), we could run:

```bash
paccache -rk2 && paccache -ruk1
```

## Cleaning yay cache

yay is an AUR helper and a pacman wrapper, a program that uses its features and some additional ones; in this case, installing packages from AUR, the community-driven repository for Arch users.

AUR does not store binary packages, but scripts called `PKGBUILD`s that download source or binary files from an external source and make the pacman package, and some extra files. yay does this automatically, but in the cache (located in `~/.cache/yay`) it stores not just the created package, but also original source or binary files and files generated in the process. We have to keep in mind that each package has its own cache directory.

yay has an operation `-Sc` similar to pacman's but, again, `paccache` seems more interesting. This script has an option, `-c` for specifying cache location, so we can write the cache directory for every package installed with yay. For example, if we want to clean yay cache keeping the two latest versions for installed packages and nothing for uninstalled packages, we could run:

```bash
paccache -rk2 -c ~/.cache/yay/*/ && paccache -ruk0 ~/.cache/yay/*/
```

However, this command only removes pacman packages, but not other files. It could be interesting to keep source or binary files from installed version (in case we upgrade the package and it uses the same ones as the previous version) and the files from the git repository of the AUR package (otherwise, yay will throw an error when we try to upgrade the package), but not the rest of the source or binary files nor the ones generated during the creation of the package.

## Bash script for cleaning both caches

**Tip:** enable `batchinstall` option for yay to avoid extra downloads when upgrading AUR packages.

First of all, install `pacman-contrib` package if you haven't done it yet:

```bash
sudo pacman -S pacman-contrib
```

I've written a bash script for easily cleaning both pacman and yay caches, based on 
[these ones](https://gist.github.com/luukvbaal/2c697b5e068471ee989bff8a56507142) I found on GitHub Gist, created by [luukvbaal](https://gist.github.com/luukvbaal).

`yaycache`

```bash
##!/usr/bin/env bash

## Assuming yay is run by user with UID 1000
admin="$(id -nu 1000)"
cachedir="/home/$admin/.cache/yay"
removed="$(comm -23 <(basename -a $(find $cachedir -mindepth 1 -maxdepth 1 -type d) | sort) <(pacman -Qqm) | xargs -r printf "$cachedir/%s\n")"

## Remove yay cache for foreign packages that are not installed anymore
rm -rf $removed

pkgcache="$(find $cachedir -mindepth 1 -maxdepth 1 -type d | xargs -r printf "-c %s\n")"

for pkgdir in "$cachedir"/*/; do

    pkgname=$(basename "$pkgdir")

    ## Remove untracked files (e. g. source/build files) excepting package files and main source files for installed version if non-git package
    if [[ ! "$pkgname" =~ ^.*-git$ ]]; then

        pkgver="$(pacman -Q $pkgname | cut -d ' ' -f2 | cut -d '-' -f1 | cut -d ':' -f2)"

        cd "$pkgdir"
        rm -f $(git ls-files --others | grep -v -e '^.*\.pkg\.tar.*$' -e '^.*/$' -e "^.*$pkgver.*$" | xargs -r printf "$pkgdir/%s\n")

    fi

    rm -rf "$pkgdir"/src/

done

## Remove everything for uninstalled foreign packages, keep latest version for uninstalled native packages, keep two latest versions for installed packages
/usr/bin/paccache -qruk0 $pkgcache
/usr/bin/paccache -qruk1
/usr/bin/paccache -qrk2 -c /var/cache/pacman/pkg $pkgcache
```

It cleans both caches using `paccache`, keeping the two latest versions for installed packages, the latest version for uninstalled packages from official repositories and nothing for uninstalled packages from AUR, it removes the directories of uninstalled packages from AUR located in yay cache and extra files from the directories of insatlled packages, excepting the files tracked by git, the pacman packages and the source or binary files of the installed version.

We can save the script, for example, in `~/.local/bin`, or whichever directory you prefer from your `$PATH`, so we can run it just typing `yaycache`.

## Pacman hook for automating cache cleaning

This script can be useful for cleaning pacman and yay caches, but we haven't reached the automation part yet.

In order to implement automation, I've written a pacman hook, a file that runs a command after installing, removing or upgrading certain packages.

In this case, the script is run after removing or upgrading any package. It does not run after installing a package, because yay installs make dependencies when needed and running the script would remove necessary files and therefore installation would fail.

`yaycache.hook`

```
[Trigger]
Operation = Upgrade
Operation = Remove
Type = Package
Target = *

[Action]
Description = Cleaning pacman and yay cache...
When = PostTransaction
Exec = /home/herbort/.local/bin/yaycache
Depends = pacman-contrib
```

Don't forget to change `Exec` path so it points to the right path where you saved the script.

We have to save the hook file in `/usr/share/libalpm/hooks`, so it gets triggered when using pacman.

## And that's all...

I hope this way of cleaning pacman and yay caches is useful for you and, of course, I appreciate any suggestions and comments.

You can find both files in this [gist](https://gist.github.com/albertored11/bfc0068f4e43ca0d7ce0af968f7314db)!
