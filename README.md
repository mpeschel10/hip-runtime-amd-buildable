# What this project is
A buildable copy of the Arch Linux package `hip-runtime-amd`, from https://github.com/archlinux/svntogit-community/tree/packages/hip-runtime-amd.

My problem is that the `python-pytorch-rocm` package in [community] gives a segfault for any operation that involves the GPU.
But I *can* run the `hip-runtime-amd` tests, so I assume my GPU is supported to some extent (It's a Radeon 6750, Navi 22 architecture).
I therefore conclude that `python-pytorch-rocm` relies on some binary that was only compiled for a subset of AMD GPUs, and I can hopefully compile it properly for my own.

My goal was to learn how to fix it myself by building a package I know does work.

# What I did
First, downloaded the source repository: https://archlinux.org/svn/
```
svn checkout --depth=empty svn://svn.archlinux.org/community
cd community
svn update hip-runtime-amd
```

Then I spent a while trying to build it and it wouldn't and I realized the version in the repository was actually older than the "current" reversion, so I don't know if the current version of hip (5.5.x) can actually build:
https://archlinux.org/packages/?sort=&q=hip-runtime-amd&maintainer=&flagged=

(In that link, as of 2023-05-11, the version in community is 5.4.3 and the version in community staging is 5.5, which tells me that I can for sure build 5.4.3 but maybe not 5.5)

Then I figured out the version number of the old PKGBUILD: `svn log --diff hip-runtime-amd/trunk/PKGBUILD` and got that version:
```sh
rm -rf hip-runtime-amd # Might be unnecessary; I don't know if svn removes unnecessary stuff
svn update hip-runtime-amd -r 1396528
```
Then I patched it by doing what the compiler said to do (adding `#include <cstdint>` to a file).

1. Before patching: `cp -r src/ROCclr-rocm-5.4.3/ src/b`
2. Apply fix: `vim src/package.new/device/devhcprintf.cpp`
3. Generate patch: `diff --unified --recursive --text src/ROCclr-rocm-5.4.3/ src/b/ > hip-amd-stdint.patch`
(I was mostly following 1. Creating Patches from https://wiki.archlinux.org/title/Patching_packages)

Then I tried to apply the patch and it just wouldn't work. I'm not sure why. Maybe I'm nuts, but it seems like `patch` only works if you are at least one directory deep?
As in, `patch -p1 < ../hip-amd-stdint.patch` will work since you're skipping the first directory with `-p`, but `patch < ./hip-amd-stdint.patch` will not at all.
Maybe I needed a `-p0` argument? Maybe there was something wrong with my directory names and omitting the first one is what did the trick? Who knows.

Finally, I added the patch to the PKGBUILD:
```sh
prepare() {
  cd "$_dirrocclr"
  patch -p2 < "$srcdir/../hip-amd-stdint.patch"
}
```
Now, this is a little hacky-- you can see that we're pulling the hip-amd-stdint.patch file from the directory above src, which is really not kosher.
But I don't want to set up hosting just for that one patch.

Anyway, I think that's everything I did. It works on my machine.

# How to compile
Ideally, just download and build.
```
git clone https://github.com/mpeschel10/hip-runtime-amd-buildable.git
makepkg
```
