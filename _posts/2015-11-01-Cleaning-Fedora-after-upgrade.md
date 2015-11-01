---
layout: post
title: "Cleaning obsolete packags after Fedora upgrade using DNF"
abstract: "With introduction of the new package management tool - DNF, some tools originally available as YUM commands or supplementary apps are now available directly as DNF switches or commands."
tags: Fedora
---

### Remove leftover packages after upgrade

Before `dnf` there was tool named `package-cleanup` that searched for various orphaned, duplicated packages. This was integrated into `dnf`.

#### Duplicates

Running `dnf --duplicated` will give you list of all packages that are duplicate on your system. However, you cannot run `dnf --quiet --duplicated | xargs -l1 dnf remove` in order to remove duplicates (`--quiet` to omit unwanted dnf info messages). When `dnf --duplicated` command finds duplicates, it prints all matches (e.g. package abc installed in versions 1 and 2, so abc-1 and abc-2 are printed).

To print out only the packages that shall be removed, use `dnf --duplicated --latest-limit -1`. The `-1` (number _minus one_) argument to `--latest-limit` says to skip 1 latest match (package abc-2 which is latest in the example).

To find and remove all duplicated packages and leave the newest in one command, fire this: `dnf --duplicated --latest-limit -1 --quiet | xargs -l1 dnf remove`

#### Old Kernels

There was `--oldkernels` switch to `package-cleanup`. In `dnf` there is no kernel-specific switch, but still it is possible to find old kernels. Running `dnf repoquery --installonly` will find such packages that are installed and installonly (which is the case for kernel packages). To have fallback option when something goes wrong with actual kernel, keeping some 3 latest kernels installed is good idea. To find kernels that could be safelly removed, and uninstall them in one turn, use `dnf repoquery --installonly --latest-limit -3 --quiet | xargs -l1 dnf remove`

#### Orphans

After upgrading, orphaned packages are those that are still installed but actually unavailable in any of enabled repositories. Typical case, after upgrading to F22, ther might remain package that is _package-version.fc21_ which will be unavailable in F22 repos. To find such packages, run `dnf list extras`

#### New broken packages

Unfortunate thisngs happen too. After upgrading, some packages might end up with unsatisfied dependencies (example of such package is _kmod-VirtualBox_ which will become broken after kernel update). Find such problems with `dnf repoquery --unsatisfied`
