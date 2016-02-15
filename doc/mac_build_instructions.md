# Mac Build Instructions

[TOC]

## Prerequisites

*   A Mac running 10.9+.
*   http://developer.apple.com/tools/xcode/XCode, 5+
*   Install
    [gclient](http://dev.chromium.org/developers/how-tos/install-depot-tools),
    part of the
    [depot_tools](http://dev.chromium.org/developers/how-tos/depottools) package
    ([download](http://dev.chromium.org/developers/how-tos/install-depot-tools)).
    gclient is a wrapper around svn that we use to manage our working copies.
*   Install Xcode's "Command Line Tools" via Xcode menu -> Preferences ->
    Downloads
*   The OSX 10.10 SDK. Run
    ```sh
    ls `xcode-select -p`/Platforms/MacOSX.platform/Developer/SDKs
    ```
    to check whether you have it. If you don't have it, you need to get one from
    an install of Xcode 6, and place it in the above directory.

## Getting the code

[Check out the source code](http://dev.chromium.org/developers/how-tos/get-the-code)
using Git. If you're new to the project, you can skip all the information about
git-svn, since you will not be committing directly to the repository.

Before checking out, go to the
[waterfall](http://build.chromium.org/buildbot/waterfall/) and check that the
source tree is open (to avoid pulling a broken tree).

The path to the build directory should not contain spaces (e.g. not
`~/Mac OS X/chromium`), as this will cause the build to fail. This includes your
drive name, the default "Macintosh HD2" for a second drive has a space.

## Building

Chromium on OS X can only be built using the [Ninja](ninja_build.md) tool and
the [Clang](clang.md) compiler. See both of those pages for further details on
how to tune the build.

Before you build, you may want to
[install API keys](https://sites.google.com/a/chromium.org/dev/developers/how-tos/api-keys)
so that Chrome-integrated Google services work. This step is optional if you
aren't testing those features.

### Raising system-wide and per-user process limits

If you see errors like the following:

```
clang: error: unable to execute command: posix_spawn failed: Resource temporarily unavailable
clang: error: clang frontend command failed due to signal (use -v to see invocation)
```

you may be running into too-low limits on the number of concurrent processes
allowed on the machine. Check:

    sysctl kern.maxproc
    sysctl kern.maxprocperuid

You can increase them with e.g.:

    sudo sysctl -w kern.maxproc=2500
    sudo sysctl -w kern.maxprocperuid=2500

But normally this shouldn't be necessary if you're building on 10.7 or higher.
If you see this, check if some rogue program spawned hundreds of processes and
kill them first.

## Faster builds

Full rebuilds are about the same speed in Debug and Release, but linking is a
lot faster in Release builds.

Run

    GYP_DEFINES=fastbuild=1 build/gyp_chromium

to disable debug symbols altogether, this makes both full rebuilds and linking
faster (at the cost of not getting symbolized backtraces in gdb).

You might also want to [install ccache](ccache_mac.md) to speed up the build.

## Running

All build output is located in the `out` directory (in the example above,
`~/chromium/src/out`).  You can find the applications at
`{Debug|Release}/ContentShell.app` and `{Debug|Release}/Chromium.app`, depending
on the selected configuration.

## Unit Tests

We have several unit test targets that build, and tests that run and pass. A
small subset of these is:

*   `unit_tests` from `chrome/chrome.gyp`
*   `base_unittests` from `base/base.gyp`
*   `net_unittests` from `net/net.gyp`
*   `url_unittests` from `url/url.gyp`

When these tests are built, you will find them in the `out/{Debug|Release}`
directory. You can run them from the command line:

    ~/chromium/src/out/Release/unit_tests


## Coding

According to the
[Chromium style guide](http://dev.chromium.org/developers/coding-style) code is
[not allowed to have whitespace on the ends of lines](http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml#Horizontal_Whitespace).
If you edit in Xcode, know that it loves adding whitespace to the ends of lines
which can make editing in Xcode more painful than it should be. The
[GTM Xcode Plugin](http://code.google.com/p/google-toolbox-for-mac/downloads/list)
adds a preference panel to Xcode that allows you to strip whitespace off of the
ends of lines on save. Documentation on how to install it is
[here](http://code.google.com/p/google-toolbox-for-mac/wiki/GTMXcodePlugin).

## Debugging

Good debugging tips can be found
[here](http://dev.chromium.org/developers/how-tos/debugging-on-os-x). If you
would like to debug in a graphical environment, rather than using `lldb` at the
command line, that is possible without building in Xcode. See
[Debugging in Xcode](http://www.chromium.org/developers/debugging-on-os-x/building-with-ninja-debugging-with-xcode)
for information on how.

## Contributing

Once you’re comfortable with building Chromium, check out
[Contributing Code](http://dev.chromium.org/developers/contributing-code) for
information about writing code for Chromium and contributing it.

## Using Xcode-Ninja Hybrid

While using Xcode is unsupported, GYP supports a hybrid approach of using ninja
for building, but Xcode for editing and driving compliation.  Xcode can still be
slow, but it runs fairly well even **with indexing enabled**.

With hybrid builds, compilation is still handled by ninja, and can be run by the
command line (e.g. ninja -C out/Debug chrome) or by choosing the chrome target
in the hybrid workspace and choosing build.

To use Xcode-Ninja Hybrid, set `GYP_GENERATORS=ninja,xcode-ninja`.

Due to the way Xcode parses ninja output paths, it's also necessary to change
the main gyp location to anything two directories deep. Otherwise Xcode build
output will not be clickable. Adding
`xcode_ninja_main_gyp=src/build/ninja/all.ninja.gyp` to your
`GYP_GENERATOR_FLAGS` will fix this.

After generating the project files with gclient runhooks, open
`src/build/ninja/all.ninja.xcworkspace`.

You may run into a problem where http://YES is opened as a new tab every time
you launch Chrome. To fix this, open the scheme editor for the Run scheme,
choose the Options tab, and uncheck "Allow debugging when using document
Versions Browser". When this option is checked, Xcode adds
`--NSDocumentRevisionsDebugMode YES` to the launch arguments, and the `YES` gets
interpreted as a URL to open.

If you want to limit the number of targets visible, which is known to improve
Xcode performance, add `xcode_ninja_executable_target_pattern=%target%` where
`%target%` is a regular expression matching executable targets you'd like to
include.

To include non-executable targets, use `xcode_ninja_target_pattern=All_iOS`.

If you have problems building, join us in `#chromium` on `irc.freenode.net` and
ask there. As mentioned above, be sure that the
[waterfall](http://build.chromium.org/buildbot/waterfall/) is green and the tree
is open before checking out. This will increase your chances of success.

## Using Emacs as `EDITOR` for `git commit`

Using the [Cocoa version of Emacs](http://emacsformacosx.com/) as the `EDITOR`
environment variable on Mac OS will cause `git commit` to open the message in a
window underneath all the others. To fix this, create a shell script somewhere
(call it `$HOME/bin/EmacsEditor` in this example) containing the following:

```
#!/bin/sh

# All of these hacks are needed to get "git commit" to launch a new
# instance of Emacs on top of everything else, properly pointing to
# the COMMIT_EDITMSG.

realpath() {
    [[ $1 = /* ]] && echo "$1" || echo "$PWD/${1#./}"
}

i=0
full_paths=()
for arg in "$@"
do
  full_paths[$i]=$(realpath $arg)
  ((++i))
done

open -nWa /Applications/Emacs.app/Contents/MacOS/Emacs --args --no-desktop \
    "${full_paths[@]}"
```

and in your `.bashrc` or similar,

    export EDITOR=$HOME/bin/EmacsEditor

## Improving performance of `git status`

`git status` is used frequently to determine the status of your checkout.  Due
to the number of files in Chromium's checkout, `git status` performance can be
quite variable.  Increasing the system's vnode cache appears to help.  By
default, this command:

    sysctl -a | egrep kern\..*vnodes

Outputs `kern.maxvnodes: 263168` (263168 is 257 * 1024).  To increase this
setting:

    sudo sysctl kern.maxvnodes=$((512*1024))

Higher values may be appropriate if you routinely move between different
Chromium checkouts.  This setting will reset on reboot, the startup setting can
be set in `/etc/sysctl.conf`:

    echo kern.maxvnodes=$((512*1024)) | sudo tee -a /etc/sysctl.conf

Or edit the file directly.

If your `git --version` reports 2.6 or higher, the following may also improve
performance of `git status`:

    git update-index --untracked-cache
