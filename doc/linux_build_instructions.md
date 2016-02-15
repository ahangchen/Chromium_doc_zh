# Linux-specific build instructions

[TOC]

## Get the code

[Get the Code](http://dev.chromium.org/developers/how-tos/get-the-code). The
general instructions on the "Get the code" page cover basic Linux build setup
and configuration.

This page documents some additional Linux-specific build issues.

## Overview

Due its complexity, Chromium uses a set of custom tools to check out and build.
Here's an overview of the steps you'll run:

1.  **gclient**. A checkout involves pulling nearly 100 different SVN
    repositories of code. This process is managed with a tool called `gclient`.
1.  **GN** / **gyp**. Cross-platform build configuration systems (GYP is the
    older one, GN is the one being transitioned to). It generates ninja build
    files. Running `gn`/`gyp` is analogous to the `./configure` step seen in
    most other software.
1.  **ninja**. The actual build itself uses `ninja`. A prebuilt binary is in
    `depot_tools` and should already be in your path if you followed the steps
    to check out Chromium.
1.  We don't provide any sort of "install" step.
1.  You may want to
    [use a chroot](using_a_linux_chroot.md) to
    isolate yourself from versioning or packaging conflicts (or to run the
    layout tests).

## Getting a checkout

[Prerequisites](linux_build_instructions_prerequisites.md): what you need before
you build.

**Note**. If you are working on Chromium OS and already have sources in
`chromiumos/chromium`, you **must** run `chrome_set_ver --runhooks` to set the
correct dependencies. This step is otherwise performed by `gclient` as part of
your checkout.

## Compilation

The weird "`src/`" directory is an artifact of `gclient`. Start with:

    $ cd src

### Faster builds

See [Linux Faster Builds](linux_faster_builds.md)

### Build every test

    $ ninja -C out/Debug

The above builds all libraries and tests in all components. **It will take
hours.**

Specifying other target names to restrict the build to just what you're
interested in. To build just the simplest unit test:

    $ ninja -C out/Debug base_unittests

### Clang builds

Information about building with Clang can be found [here](clang.md).

### Output

Executables are written in `src/out/Debug/` for Debug builds, and
`src/out/Release/` for Release builds.

### Release mode

Pass `-C out/Release` to the ninja invocation:

    $ ninja -C out/Release chrome

### Seeing the commands

If you want to see the actual commands that ninja is invoking, add `-v` to the
ninja invocation.

    $ ninja -v -C out/Debug chrome

This is useful if, for example, you are debugging gyp changes, or otherwise need
to see what ninja is actually doing.

### Clean builds

If you're using GN, you can clean the build directory (`out/Default` in this
example):

    gn clean out/Default

This will delete all files except a bootstrap ninja file necessary for
recreating the build.

If you're using GYP, do:

    rm -rf out
    gclient runhooks

Ninja can also be used to clean a build with `ninja -C out/Debug -t clean` but
this will not be as complete as the above methods.

### Linker Crashes

If, during the final link stage:

    LINK(target) out/Debug/chrome

You get an error like:

```
collect2: ld terminated with signal 6 Aborted terminate called after throwing an
instance of 'std::bad_alloc'

collect2: ld terminated with signal 11 [Segmentation fault], core dumped
```
you are probably running out of memory when linking.  Try one of:

1.  Use the `gold` linker
1.  Build on a 64-bit computer
1.  Build in Release mode (debugging symbols require a lot of memory)
1.  Build as shared libraries (note: this build is for developers only, and may
    have broken functionality)

Most of these are described on the [Linux Faster Builds](linux_faster_builds.md)
page.

## Advanced Features

*   Building frequently? See [LinuxFasterBuilds](linux_faster_builds.md).
*   Cross-compiling for ARM? See [LinuxChromiumArm](linux_chromium_arm.md).
*   Want to use Eclipse as your IDE? See
    [LinuxEclipseDev](linux_eclipse_dev.md).
*   Built version as Default Browser? See
    [LinuxDevBuildAsDefaultBrowser](linux_dev_build_as_default_browser.md).

## Next Steps

If you want to contribute to the effort toward a Chromium-based browser for
Linux, please check out the [Linux Development page](linux_development.md) for
more information.
