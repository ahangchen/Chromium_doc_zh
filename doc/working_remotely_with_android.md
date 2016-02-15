# Working remotely with Android

[TOC]


## Introduction

When you call /build/android/run_tests.py or
/build/android/run_instrumentation_tests.py it assumes an android device
is attached to the local host.

TODO: these scripts do not exist.

If you want to work remotely from your laptop with an android device attached to
it, while keeping an ssh connection to a remote desktop machine where you have
your build environment setup, you will have to use one of the two alternatives
listed below.

## Option 1: SSHFS - Mounting the out/Debug directory

### On your remote host machine

You can open a regular ssh to your host.

    # build it
    desktop$ cd $SRC;
    desktop$ . build/android/envsetup.sh
    desktop$ build/gyp_chromium -DOS=android
    desktop$ ninja -C out/Debug

See also
[Android Build Instructions](https://www.chromium.org/developers/how-tos/android-build-instructions).

### On your laptop

You have to have an android device attached to it.

```shell
# Install sshfs

laptop$ sudo apt-get install sshfs

# Mount the chrome source from your remote host machine into your local laptop.

laptop$ mkdir ~/chrome_sshfs
laptop$ sshfs your.host.machine:/usr/local/code/chrome/src ./chrome_sshfs

# Setup enviroment.

laptop$ cd chrome_sshfs
laptop$ . build/android/envsetup.sh
laptop$ adb devices
laptop$ adb root

# Install APK (which was previously built in the host machine).

laptop$ python build/android/adb_install_apk.py --apk ContentShell.apk --apk_package org.chromium.content_shell

# Run tests.

laptop$ python build/android/run_instrumentation_tests.py -I --test-apk ContentShellTest -vvv
```

*** note
This is assuming you have the exact same linux version on your host machine and
in your laptop.
***

But if you have different versions, lets say, ubuntu lucid on your laptop, and the newer ubuntu precise on your host machine, some binaries compiled on the host will not work on your laptop.
In this case you will have to recompile these binaries in your laptop:

```shell
# May need to install dependencies on your laptop.

laptop$ sudo ./build/install-build-deps-android.sh

# Rebuild the needed binaries on your laptop.

laptop$ build/gyp_chromium -DOS=android
laptop$ ninja -C out/Debug md5sum host_forwarder
```

## Option 2: SSH Tunneling

### Option 2a: Use a script

Copy /tools/android/adb_remote_setup.sh to your laptop, then run it.
adb_remote_setup.sh updates itself, so you only need to copy it once.

```shell
laptop$ curl -sSf "https://chromium.googlesource.com/chromium/src.git/+/master/tools/android/adb_remote_setup.sh?format=TEXT | base64 --decode > adb_remote_setup.sh
laptop$ chmod +x adb_remote_setup.sh
laptop$ ./adb_remote_setup.sh <desktop_hostname> <path_to_adb_on_desktop>
```

### Option 2b: Manual tunneling

You have to make sure that ports 5037, 10000, ad 10201 are not being used on
either your laptop or your desktop. Try the command: `netstat -nap | grep 10000`
to see

Kill the pids that are using those ports.

#### On your host machine

```shell
desktop$ killall adb
desktop$ killall host_forwarder
```

#### On your laptop

```shell
laptop$ ssh -C -R 5037:localhost:5037 -R 10000:localhost:10000 -R 10201:localhost:10201 <desktop_host_name>
```

#### On your host machine

```shell
desktop$ python build/android/run_instrumentation_tests.py -I --test-apk ContentShellTest -vvv
```
