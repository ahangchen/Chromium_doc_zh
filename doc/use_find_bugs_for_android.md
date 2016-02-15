# Use FindBugs for Android

[FindBugs](http://findbugs.sourceforge.net) is an open source static analysis
tool from the University of Maryland that looks for potential bugs in Java class
files. We have some scripts to run it over the Java code at build time.

## How To Run

For gyp builds, add `run_findbugs=1` to your `GYP_DEFINES`.

For gn builds, add `run_findbugs=true` to the args you pass to `gn gen`:

    gn gen --args='target_os="android" run_findbugs=true'

Note that running findbugs will add time to your build. The amount of additional
time required depends on the number of targets on which findbugs runs, though it
will usually be between 1-10 minutes.

Some of the warnings are false positives. In general, they should be suppressed
using
[@SuppressFBWarnings](https://code.google.com/p/chromium/codesearch#chromium/src/base/android/java/src/org/chromium/base/annotations/SuppressFBWarnings.java).
In the rare event that a warning should be suppressed across the entire
code base, it should be added to the
[exclusion file](https://code.google.com/p/chromium/codesearch#chromium/src/build/android/findbugs_filter/findbugs_exclude.xml)
instead. If you modify this file:

*   Include a comment that says what you're suppressing and why.
*   The existing suppressions should give you an idea of the syntax. See also
    the FindBugs documentation. Note that the documentation doesn't seem totally
    accurate (there's probably some version skew between the online docs and the
    version of FindBugs we're using) so you may have to experiment a little.

# Chromium's [FindBugs](http://findbugs.sourceforge.net) plugin

We have
[FindBugs plugin](https://code.google.com/p/chromium/codesearch#chromium/src/tools/android/findbugs_plugin/)
to enforce chromium specific Java rules. It currently detects:

*   Synchronized method
*   Synchronized this

# [FindBugs](http://findbugs.sourceforge.net) on the Bots

[FindBugs](http://findbugs.sourceforge.net) is configured to run on:

*   [android_clang_dbg_recipe](http://build.chromium.org/p/tryserver.chromium.linux/builders/android_clang_dbg_recipe)
    on the commit queue
*   [Android Clang Builder (dbg)](http://build.chromium.org/p/chromium.linux/builders/Android%20Clang%20Builder%20\(dbg\))
    on the main waterfall
