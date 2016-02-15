# Common Build Tasks

The Chromium build system is a complicated beast of a system, and it is not very
well documented beyond the basics of getting the source and building the
Chromium product. This page has more advanced information about the build
system.

If you're new to Chromium development, read the
[getting started guides](http://dev.chromium.org/developers/how-tos/get-the-code).

There is some additional documentation on
[setting GYP build parameters](http://dev.chromium.org/developers/gyp-environment-variables).

[TOC]

## Faster Builds

### Components Build

A non-standard build configuration is to use dynamic linking instead of static
linking for the various modules in the Chromium codebase. This results in
significantly faster link times, but is a divergence from what is shipped and
primarily tested. To enable the
[component build](http://www.chromium.org/developers/how-tos/component-build):

    $ GYP_DEFINES="component=shared_library" gclient runhooks

or

```
C:\...\src>set GYP_DEFINES=component=shared_library
C:\...\src>gclient runhooks
```

### Windows: Debug Builds Link Faster

On Windows if using the components build, building in debug mode will generally
link faster. This is because in debug mode, the linker works incrementally. In
release mode, a full link is performed each time.

### Mac: Disable Release Mode Stripping

On Mac, if building in release mode, one of the final build steps will be to
strip the build products and create dSYM files. This process can slow down your
incremental builds, but it can be disabled with the following define:

    $ GYP_DEFINES="mac_strip_release=0" gclient runhooks

### Mac: DCHECKs in Release Mode

DCHECKs are only designed to be run in debug builds. But building in release
mode on Mac is significantly faster. You can have your cake and eat it too by
building release mode with DCHECKs enabled using the following define:

    $ GYP_DEFINES="dcheck_always_on=1" gclient runhooks

### Linux

Linux has its own page on [making the build faster](linux_faster_builds.md).

## Configuring the Build

### Environment Variables

There are various environment variables that can be passed to the metabuild
system GYP when generating project files. This is a summary of them:

TODO(andybons): Convert to list.

|:-------------|:--------------------------------------------------------------|
| `GYP_DEFINES` | A set of key=value pairs separated by space that will set default values of variables used in .gyp and .gypi files |
| `GYP_GENERATORS` | The specific generator that creates build-system specific files |
| `GYP_GENERATOR_FLAGS` | Flags that are passed down to the tool that generates the build-system specific files |
| `GYP_GENERATOR_OUTPUT` | The directory that the top-level build output directory is relative to |

Note also that GYP uses CPPFLAGS, CFLAGS, and CXXFLAGS when generating ninja
files (the values at build time = ninja run time are _not_ used); see
[gyp/generator/ninja.py](https://code.google.com/p/chromium/codesearch#chromium/src/tools/gyp/pylib/gyp/generator/ninja.py&q=cxxflags).

### Variable Files

If you want to keep a set of variables established, there are a couple of magic
files that GYP reads:

#### chromium.gyp\_env

Next to your top-level `/src/` directory, create a file called
`chromium.gyp_env`. This holds a JSON dictionary, with the keys being any of the
above environment variables. For the full list of supported keys, see
[/src/build/gyp_helper.py](/build/gyp_helper.py).

```
{
  'variables': {
    'mac_strip_release': 0,
  }, 'GYP_DEFINES':
    'clang=1 ' 'component=shared_library ' 'dcheck_always_on=1 '
}
```

#### include.gyp

Or globally in your home directory, create a file `~/.gyp/include.gypi`.

#### supplement.gypi

The build system will also include any files named `/src/*/supplement.gypi`,
which should be in the same format as include.gyp above.

### Change the Build System

Most platforms support multiple build systems (Windows: different Visual Studios
versions and ninja, Mac: Xcode and ninja, etc.). A sensible default is selected,
but it can be overridden:

    $ GYP_GENERATORS=ninja gclient runhooks

[Ninja](ninja_build.md) is generally the fastest way to build anything on any
platform.

### Change Build Output Directory

If you need to change a compile-time flag and do not want to touch your current
build output, you can re-run GYP and place output into a new directory, like so,
assuming you are using ninja:

```shell
$ GYP_GENERATOR_FLAGS="output_dir=out_other_ninja" gclient runhooks
$ ninja -C out_other_ninja/Release chrome
```

Alternatively, you can do the following, which should work with all GYP
generators, but the out directory is nested as `out_other/out/`.

```shell
$ GYP_GENERATOR_OUTPUT="out_other" gclient runhooks
$ ninja -C out_other/out/Release chrome
```

**Note:** If you wish to run the WebKit layout tests, make sure you specify the
new directory using `--build-directory=out_other_ninja` (don't include the
`Release` part).

### Building Google Chrome

To build Chrome, you need to be a Google employee and have access to the
[src-internal](https://goto.google.com/src-internal) repository. Once your
checkout is set up, you can run gclient like so:

    $ GYP_DEFINES="branding=Chrome buildtype=Official" gclient runhooks

Then building the `chrome` target will produce the official build. This tip can
be used in conjunction with changing the output directory, since changing these
defines will rebuild the world.

Also note that some GYP\_DEFINES flags are incompatible with the official build.
If you get an error when you try to build, try removing all your flags and start
with just the above ones.
