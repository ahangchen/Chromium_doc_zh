# Ninja Build

Ninja is a build system written with the specific goal of improving the
edit-compile cycle time. It is used by default everywhere except when building
for iOS.

Ninja behaves very similar to Make -- the major feature is that it starts
building files nearly instantly. (It has a number of minor user interface
improvements to make as well.)

Read more about Ninja at
[the Ninja home page](http://martine.github.com/ninja/).

## Using it

### Configure your system to use Ninja

#### Install

Ninja is included in `depot_tools` as well as `gyp`, so there's nothing to
install.

## Build instructions

To build Chrome:

    cd /path/to/chrome/src
    ninja -C out/Debug chrome

Specify `out/Release` for a release build. I recommend setting up an alias so
that you don't need to type out that build directory path.

If you want to build all targets, use `ninja -C out/Debug all`. It's faster to
build only the target you're working on, like `chrome` or `unit_tests`.

## Android

Identical to Linux, just make sure `OS=android` is in your `GYP_DEFINES`. You
want to build one of the apk targets, e.g. `content_shell_apk`.

## Windows

Similar to Linux. It uses MSVS's `cl.exe`, `link.exe`, etc. so you still need to
have VS installed. To use it, open `cmd.exe`, go to your chrome checkout, and
run:

    set GYP_DEFINES=component=shared_library
    python build\gyp_chromium
    ninja -C out\Debug chrome.exe

`component=shared_library` is optional but recommended for faster links.

You can also set `GYP_GENERATORS=ninja,msvs-ninja` to get both VS projects
generated if you want to use VS just to browse/edit (but then gyp takes twice as
long to run).

If you're using Express or the Windows SDK by itself (rather than using a Visual
Studio install), you'll need to run from a vcvarsall command prompt.

### Debugging

Miss VS for debugging?

```
devenv.com /debugexe chrome.exe --my-great-args "go here" --single-process etc
```

Miss Xcode for debugging? Read
http://dev.chromium.org/developers/debugging-on-os-x/building-with-ninja-debugging-with-xcode

### Without Visual Studio

That is, building with just the WinDDK. This is documented in the
[regular build instructions](http://dev.chromium.org/developers/how-tos/build-instructions-windows#TOC-Setting-up-the-environment-for-building-with-Visual-C-2010-Express-or-Windows-7.1-SDK).

## Tweaks

### Building through errors

Pass a flag like `-k3` to make Ninja build until it hits three errors instead of
stopping at the first.

### Parallelism

Pass a flag like `-j8` to use 8 parallel processes, or `-j1` to compile just one
at a time (helpful if you're getting weird compiler errors). By default Ninja
tries to use all your processors.

### More options

There are more options. Run `ninja --help` to see them all.

### Custom build configs

You can write a specific build config to a specific output directory via the
`-G` flags to gyp. Here's an example from jamesr:
`build/gyp_chromium -Gconfig=Release -Goutput_dir=out_profiling -Dprofiling=1
-Dlinux_fpic=0`

## Bugs

If you encounter any problems, please file a bug at http://crbug.com/new with
label `ninja` and cc `thakis@` or `scottmg@`.  Assume that it is a bug in Ninja
before you bother anyone about e.g. link problems.
