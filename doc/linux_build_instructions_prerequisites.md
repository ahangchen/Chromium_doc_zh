# Linux Build Instructions — Prerequisites

This page describes system requirements for building Chromium on Linux.

[TOC]

## System Requirements

### Linux distribution

You should be able to build Chromium on any reasonably modern Linux
distribution, but there are a lot of distributions and we sometimes break things
on one or another. Internally, our development platform has been a variant of
Ubuntu 14.04 (Trusty Tahr); we expect you will have the most luck on this
platform, although directions for other popular platforms are included below.

### Disk space

It takes about 10GB or so of disk space to check out and build the source tree.
This number grows over time.

### Memory space

It takes about 8GB of swap file to link chromium and its tests. If you get an
out-of-memory error during the final link, you will need to add swap space with
swapon. It's recommended to have at least 4GB of memory available for building a
statically linked debug build. Dynamic linking and/or building a release build
lowers memory requirements. People with less than 8GB of memory may want to not
build tests since they are quite large.

### 64-bit Systems

Chromium can be compiled as either a 32-bit or 64-bit application. Chromium
requires several system libraries to compile and run. While it is possible to
compile and run a 32-bit Chromium on 64-bit Linux, many distributions are
missing the necessary 32-bit libraries, and will result in build or run-time
errors.

### Depot tools

Before setting up the environment, make sure you install the
[depot tools](http://dev.chromium.org/developers/how-tos/depottools) first.

## Software Requirements

### Ubuntu Setup

Run [build/install-build-deps.sh](/build/install-build-deps.sh) The script only
supports current releases as listed on https://wiki.ubuntu.com/Releases.

Building on Linux requires software not usually installed with the
distributions.

The script attempts to automate installing the required software. This script is
used to set up the canonical builders, and as such is the most up to date
reference for the required prerequisites.

### Other distributions

Note: Other distributions are not officially supported for building and the
instructions below might be outdated.

#### Debian Setup

Follow the Ubuntu instructions above.

If you want to install the build-deps manually, note that the original packages
are for Ubuntu. Here are the Debian equivalents:

*   libexpat-dev -> libexpat1-dev
*   freetype-dev -> libfreetype6-dev
*   libbzip2-dev -> libbz2-dev
*   libcupsys2-dev -> libcups2-dev

Additionally, if you're building Chromium components for Android, you'll need to
install the package: lib32z1

#### openSUSE Setup

For openSUSE 11.0 and later, see
[Linux openSUSE Build Instructions](linux_open_suse_build_instructions.md).

#### Fedora Setup

Recent systems:

```shell
su -c 'yum install subversion pkgconfig python perl gcc-c++ bison \
flex gperf nss-devel nspr-devel gtk2-devel glib2-devel freetype-devel \
atk-devel pango-devel cairo-devel fontconfig-devel GConf2-devel \
dbus-devel alsa-lib-devel libX11-devel expat-devel bzip2-devel \
dbus-glib-devel elfutils-libelf-devel libjpeg-devel \
mesa-libGLU-devel libXScrnSaver-devel \
libgnome-keyring-devel cups-devel libXtst-devel libXt-devel pam-devel'
```

The msttcorefonts packages can be obtained by following the instructions present
here: http://www.fedorafaq.org/#installfonts

For the optional packages:

*   php-cgi is provided by the php-cli package
*   wdiff doesn't exist in Fedora repositories, a possible alternative would be
    dwdiff
*   sun-java6-fonts doesn't exist in Fedora repositories, needs investigating

    su -c 'yum install httpd mod_ssl php php-cli wdiff'

#### Arch Linux Setup

Most of these packages are probably already installed since they're often used,
and the parameter --needed ensures that packages up to date are not reinstalled.

```shell
sudo pacman -S --needed python perl gcc gcc-libs bison flex gperf pkgconfig \
  nss alsa-lib gconf glib2 gtk2 nspr ttf-ms-fonts freetype2 cairo dbus \
  libgnome-keyring
```

For the optional packages on Arch Linux:

*   php-cgi is provided with pacman
*   wdiff is not in the main repository but dwdiff is. You can get wdiff in
    AUR/yaourt
*   sun-java6-fonts do not seem to be in main repository or AUR.

For a successful build, add `'remove_webcore_debug_symbols': 1,` to the
variables-object in include.gypi. Tested on 64-bit Arch Linux.

TODO: Figure out how to make it build with the WebCore debug symbols. `make V=1`
can be useful for solving the problem.

#### Mandriva setup

```shell
urpmi lib64fontconfig-devel lib64alsa2-devel lib64dbus-1-devel \
lib64GConf2-devel lib64freetype6-devel lib64atk1.0-devel lib64gtk+2.0_0-devel \
lib64pango1.0-devel lib64cairo-devel lib64nss-devel lib64nspr-devel g++ python \
perl bison flex subversion gperf
```

*** note
Note 1: msttcorefonts are not available, you will need to build your own (see
instructions, not hard to do, see
[mandriva_msttcorefonts.md](mandriva_msttcorefonts.md)) or use drakfont to
import the fonts from a windows installation
***

*** note
Note 2: these packages are for 64 bit, to download the 32 bit packages,
substitute lib64 with lib
***

*** note
Note 3: some of these packages might not be explicitly necessary as they come as
dependencies, there is no harm in including them however.
***

*** note
Note 4: to build on 64 bit systems use, instead of
`GYP_DEFINES=target_arch=x64`, as mentioned in the general notes for building on
64 bit:

```shell
export GYP_DEFINES="target_arch=x64"
gclient runhooks --force
```
***

#### Gentoo setup

    emerge www-client/chromium
