# The Clang Static Analyzer

See the [official clang static analyzer page](http://clang-analyzer.llvm.org/)
for background.

We don't run this regularly (because the analyzer's
[support for C++ isn't great yet](http://clang-analyzer.llvm.org/dev_cxx.html)),
so everything on this page is likely broken. The last time I checked, the
analyzer reported mostly uninteresting things. This assumes you're
[building chromium with clang](clang.md).

You need an llvm checkout to get `scan-build` and `scan-view`; the easiest way
to get that is to run

```shell
tools/clang/scripts/update.py --force-local-build --without-android
```

## With make

To build base, if you use the make build:

```
builddir_name=out_analyze \
PATH=$PWD/third_party/llvm-build/Release+Asserts/bin:$PATH  \
third_party/llvm/tools/clang/tools/scan-build/scan-build  \
    --keep-going --use-cc clang --use-c++ clang++ \
    make -j8 base
```

(`builddir_name` is set to force a clobber build.)

Once that's done, run `third_party/llvm/tools/clang/tools/scan-view/scan-view`
to see the results; pass in the pass that `scan-build` outputs.

## With ninja

scan-build does its stuff by mucking with $CC/$CXX, which ninja ignores. gyp
does look at $CC/$CXX however, so you need to first run gyp\_chromium under
scan-build:

```shell
time GYP_GENERATORS=ninja \
GYP_DEFINES='component=shared_library clang_use_chrome_plugins=0 \
    mac_strip_release=0 dcheck_always_on=1' \
third_party/llvm/tools/clang/tools/scan-build/scan-build \
    --use-analyzer $PWD/third_party/llvm-build/Release+Asserts/bin/clang \
    build/gyp_chromium -Goutput_dir=out_analyze
```

You then need to run the build under scan-build too, to get a HTML report:

```shell
time third_party/llvm/tools/clang/tools/scan-build/scan-build \
    --use-analyzer $PWD/third_party/llvm-build/Release+Asserts/bin/clang \
    ninja -C out_analyze/Release/ base
```

Then run `scan-view` as described above.

## Known False Positives

* http://llvm.org/bugs/show_bug.cgi?id=11425

## Stuff found by the static analyzer

*   https://code.google.com/p/skia/issues/detail?id=399
*   https://code.google.com/p/skia/issues/detail?id=400
*   https://codereview.chromium.org/8308008/
*   https://codereview.chromium.org/8313008/
*   https://codereview.chromium.org/8308009/
*   https://codereview.chromium.org/10031018/
*   https://codereview.chromium.org/12390058/
