# Code Coverage

## Categories of coverage

*   executed - this line of code was hit during execution
*   instrumented - this line of code was part of the compilation unit, but not
    executed
*   missing - in a source file, but not compiled.
*   ignored - not an executable line, or a line we don't care about

Coverage is calculated as `exe / (inst + miss)`. In general, lines that are in
`miss` should be ignored, but our exclusion rules are not good enough.

## Buildbots

Buildbots are currently on the
[experimental waterfall](http://build.chromium.org/buildbot/waterfall.fyi/waterfall).
The coverage figures they calculate come from running some subset of the
chromium testing suite.

*   [Linux](http://build.chromium.org/buildbot/waterfall.fyi/builders/Linux%20Coverage%20(dbg))
    - uses `gcov`
*   [Windows](http://build.chromium.org/buildbot/waterfall.fyi/builders/Win%20Coverage%20%28dbg%29)
*   [Mac](http://build.chromium.org/buildbot/waterfall.fyi/builders/Mac%20Coverage%20%28dbg%29)

Also,

*   [Coverage dashboard](http://build.chromium.org/buildbot/coverage/)
*   [Example coverage summary](http://build.chromium.org/buildbot/coverage/linux-debug/49936/)
    - the coverage is calculated at directory and file level, and the directory
    structure is navigable via the **Subdirectories** table.

## Calculating coverage locally

TODO

## Advanced Tips

Sometimes a line of code should never be reached (e.g., `NOTREACHED()`). These
can be marked in the source with `// COV_NF_LINE`. Note that this syntax is
exact.
