# Tips for improving build speed on Linux

This list is sorted such that the largest speedup is first; see
[Linux build instructions](linux_build_instructions.md) for context and
[Faster Builds](common_build_tasks.md) for non-Linux-specific techniques.

[TOC]

## Use goma

If you work at Google, you can use goma for distributed builds; this is similar
to [distcc](http://en.wikipedia.org/wiki/Distcc). See [go/ma](http://go/ma) for
documentation.

Even without goma, you can do distributed builds with distcc (if you have access
to other machines), or a parallel build locally if have multiple cores.

Whether using goma, distcc, or parallel building, you can specify the number of
build processes with `-jX` where `X` is the number of processes to start.

## Use Icecc

[Icecc](https://github.com/icecc/icecream) is the distributed compiler with a
central scheduler to share build load. Currently, many external contributors use
it. e.g. Intel, Opera, Samsung.

When you use Icecc, you need to set some gyp variables.

    linux_use_bundled_binutils=0**

`-B` option is not supported.
[relevant commit](https://github.com/icecc/icecream/commit/b2ce5b9cc4bd1900f55c3684214e409fa81e7a92)

    linux_use_debug_fission=0

[debug fission](http://gcc.gnu.org/wiki/DebugFission) is not supported.
[bug](https://github.com/icecc/icecream/issues/86)

    clang=0

Icecc doesn't support clang yet.

    use_sysroot=0

Icecc doesn't work with sysroot.

    linux_use_bundled_gold=0

Using the system linker is necessary when using glibc 2.21 or newer. See
[related bug](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=808181).

## Build only specific targets

If you specify just the target(s) you want built, the build will only walk that
portion of the dependency graph:

    cd $CHROMIUM_ROOT/src
    ninja -C out/Debug base_unittests

## Linking

### Dynamically link

We normally statically link everything into one final executable, which produces
enormous (nearly 1gb in debug mode) files. If you dynamically link, you save a
lot of time linking for a bit of time during startup, which is fine especially
when you're in an edit/compile/test cycle.

Run gyp with the `-Dcomponent=shared_library` flag to put it in this
configuration.  (Or set those flags via the `GYP_DEFINES` environment variable.)

e.g.

    build/gyp_chromium -D component=shared_library
    ninja -C out/Debug chrome

See the
[component build page](http://www.chromium.org/developers/how-tos/component-build)
for more information.

### Linking using gold

The experimental "gold" linker is much faster than the standard BFD linker.

On some systems (including Debian experimental, Ubuntu Karmic and beyond), there
exists a `binutils-gold` package. Do not install this version! Having gold as
the default linker is known to break kernel / kernel module builds.

The Chrome tree now includes a binary of gold compiled for x64 Linux. It is used
by default on those systems.

On other systems, to safely install gold, make sure the final binary is named
`ld` and then set `CC/CXX` appropriately, e.g.
`export CC="gcc -B/usr/local/gold/bin"` and similarly for `CXX`. Alternatively,
you can add `/usr/local/gold/bin` to your `PATH` in front of `/usr/bin`.

## WebKit

### Build WebKit without debug symbols

WebKit is about half our weight in terms of debug symbols. (Lots of templates!)
If you're working on UI bits where you don't care to trace into WebKit you can
cut down the size and slowness of debug builds significantly by building WebKit
without debug symbols.

Set the gyp variable `remove_webcore_debug_symbols=1`, either via the
`GYP_DEFINES` environment variable, the `-D` flag to gyp, or by adding the
following to `~/.gyp/include.gypi`:

```
{
  'variables': {
    'remove_webcore_debug_symbols': 1,
  },
}
```

## Tune ccache for multiple working directories

(Ignore this if you use goma.)

Increase your ccache hit rate by setting `CCACHE_BASEDIR` to a parent directory
that the working directories all have in common (e.g.,
`/home/yourusername/development`). Consider using
`CCACHE_SLOPPINESS=include_file_mtime` (since if you are using multiple working
directories, header times in svn sync'ed portions of your trees will be
different - see
[the ccache troubleshooting section](http://ccache.samba.org/manual.html#_troubleshooting)
for additional information). If you use symbolic links from your home directory
to get to the local physical disk directory where you keep those working
development directories, consider putting

    alias cd="cd -P"

in your `.bashrc` so that `$PWD` or `cwd` always refers to a physical, not
logical directory (and make sure `CCACHE_BASEDIR` also refers to a physical
parent).

If you tune ccache correctly, a second working directory that uses a branch
tracking trunk and is up-to-date with trunk and was gclient sync'ed at about the
same time should build chrome in about 1/3 the time, and the cache misses as
reported by `ccache -s` should barely increase.

This is especially useful if you use `git-new-workdir` and keep multiple local
working directories going at once.

## Using tmpfs

You can use tmpfs for the build output to reduce the amount of disk writes
required. I.e. mount tmpfs to the output directory where the build output goes:

As root:

    mount -t tmpfs -o size=20G,nr_inodes=40k,mode=1777 tmpfs /path/to/out

*** note
**Caveat:** You need to have enough RAM + swap to back the tmpfs. For a full
debug build, you will need about 20 GB. Less for just building the chrome target
or for a release build.
***

Quick and dirty benchmark numbers on a HP Z600 (Intel core i7, 16 cores
hyperthreaded, 12 GB RAM)

*   With tmpfs:
    *   12m:20s
*   Without tmpfs
    *   15m:40s
