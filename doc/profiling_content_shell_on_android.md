# Profiling Content Shell on Android

Below are the instructions for setting up profiling for Content Shell on
Android. This will let you generate profiles for ContentShell. This will require
linux, building an userdebug Android build, and wiping the device.

[TOC]

## Prepare your device.

You need an Android 4.2+ device (Galaxy Nexus, Nexus 4, 7, 10, etc.) which you
don’t mind erasing all data, rooting, and installing a userdebug build on.

## Get and build `content_shell_apk` for Android

(These instructions have been carefully distilled from the
[Android Build Instructions](android_build_instructions.md).)

1.  Get the code! You’ll want a second checkout as this will be
    android-specific. You know the drill:
    http://dev.chromium.org/developers/how-tos/get-the-code
1.  Append this to your `.gclient` file: `target_os = ['android']`
1.  Create `chromium.gyp_env` next to your `.gclient` file:
    `echo "{ 'GYP_DEFINES': 'OS=android', }" > chromium.gyp_env`
1.  (Note: All these scripts assume you’re using "bash" (default) as your
    shell.)
1.  Sync and runhooks (be careful not to run hooks on the first sync):

    ```
    gclient sync --nohooks
    . build/android/envsetup.sh
    gclient runhooks
    ```

1.  No need to install any API Keys.
1.  Install Oracle’s Java: http://goo.gl/uPRSq. Grab the appropriate x64 .bin
    file, `chmod +x`, and then execute to extract. You then move that extracted
    tree into /usr/lib/jvm/, rename it java-6-sun and set:

    ```
    export JAVA_HOME=/usr/lib/jvm/java-6-sun
    export ANDROID_JAVA_HOME=/usr/lib/jvm/java-6-sun
    ```

1.  Type ‘`java -version`’ and make sure it says java version `1.6.0_35` without
    any mention of openjdk before proceeding.
1.  `sudo build/install-build-deps-android.sh`
1.  Time to build!

    ```
    ninja -C out/Release content_shell_apk
    ```

## Setup the physical device

Plug in your device. Make sure you can talk to your device, try "`adb shell ls`"

## Root your device and install a userdebug build

1.  This may require building your own version of Android:
    http://source.android.com/source/building-devices.html
1.  A build that works is: `manta / android-4.2.2_r1` or
    `master / full_manta-userdebug`.

## Root your device

1.  Run `adb root`. Every time you connect your device you’ll want to run this.
1.  If adb is not available, make sure to run `. build/android/envsetup.sh`

If you get the error `error: device offline`, you may need to become a developer
on your device before Linux will see it. On Jellybean 4.2.1 and above this
requires going to “about phone” or “about tablet” and clicking the build number
7 times:
http://androidmuscle.com/how-to-enable-usb-debugging-developer-options-on-nexus-4-and-android-4-2-devices/

## Run a Telemetry perf profiler

You can run any Telemetry benchmark with `--profiler=perf`, and it will:

1.  Download `perf` and `perfhost`
1.  Install on your device
1.  Run the test
1.  Setup symlinks to work with the `--symfs` parameter

You can also run "manual" tests with Telemetry, more information here:
http://www.chromium.org/developers/telemetry/profiling#TOC-Manual-Profiling---Android

The following steps describe building `perf`, which is no longer necessary if
you use Telemetry.

## Install `/system/bin/perf` on your device (not needed for Telemetry)

    # From inside the android source tree (not inside Chromium)
    mmm external/linux-tools-perf/
    adb remount # (allows you to write to the system image)
    adb sync
    adb shell perf top # check that perf can get samples (don’t expect symbols)

## Enable profiling

Rebuild `content_shell_apk` with profiling enabled

    export GYP_DEFINES="$GYP_DEFINES profiling=1"
    build/gyp_chromium
    ninja -C out/Release content_shell_apk

## Install ContentShell

Install with the following:

    build/android/adb_install_apk.py \
        --apk out/Release/apks/ContentShell.apk \
        --apk_package org.chromium.content_shell

## Run ContentShell

Run with the following:

    ./build/android/adb_run_content_shell

If `content_shell` “stopped unexpectedly” use `adb logcat` to debug. If you see
ResourceExtractor exceptions, a clean build is your solution.
https://crbug.com/164220

## Setup a `symbols` directory with symbols from your build (not needed for Telemetry)

1.  Figure out exactly what path `content_shell_apk` (or chrome, etc) installs
    to.
    *   On the device, navigate ContentShell to about:crash


    adb logcat | grep libcontent_shell_content_view.so

You should find a path that’s something like
`/data/app-lib/org.chromium.content_shell-1/libcontent_shell_content_view.so`

1.  Make a symbols directory
    ```
    mkdir symbols (this guide assumes you put this next to src/)
    ```
1.  Make a symlink from your symbols directory to your un-stripped
    `content_shell`.

    ```
    # Use whatever path in app-lib you got above
    mkdir -p symbols/data/app-lib/org.chromium.content_shell-1
    ln -s `pwd`/src/out/Release/lib/libcontent_shell_content_view.so \
        `pwd`/symbols/data/app-lib/org.chromium.content_shell-1
    ```

## Install `perfhost_linux` locally (not needed for Telemetry)

Note: modern versions of perf may also be able to process the perf.data files
from the device.

1.  `perfhost_linux` can be built from:
    https://android.googlesource.com/platform/external/linux-tools-perf/.
1.  Place `perfhost_linux` next to symbols, src, etc.

    chmod a+x perfhost_linux

## Actually record a profile on the device!

Run the following:

    adb shell ps | grep content (look for the pid of the sandboxed_process)
    adb shell perf record -g -p 12345 sleep 5
    adb pull /data/perf.data


## Create the report

1.  Run the following:

    ```
    ./perfhost_linux report -g -i perf.data --symfs symbols/
    ```

1.  If you don’t see chromium/webkit symbols, make sure that you built/pushed
    Release, and that the symlink you created to the .so is valid!
1.  If you have symbols, but your callstacks are nonsense, make sure you ran
    `build/gyp_chromium` after setting `profiling=1`, and rebuilt.

## Add symbols for the kernel

1.  By default, /proc/kallsyms returns 0 for all symbols, to fix this, set
    `/proc/sys/kernel/kptr_restrict` to `0`:

    ```
    adb shell echo “0” > /proc/sys/kernel/kptr_restrict
    ```

1.  See http://lwn.net/Articles/420403/ for explanation of what this does.

    ```
    adb pull /proc/kallsyms symbols/kallsyms
    ```

1.  Now add --kallsyms to your perfhost\_linux command:
    ```
    ./perfhost_linux report -g -i perf.data --symfs symbols/ \
        --kallsyms=symbols/kallsyms
    ```
