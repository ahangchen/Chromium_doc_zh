# Windows incremental linking

Include in your `GYP_DEFINES`: `incremental_chrome_dll=1`. This turns on the
equivalent of Use Library Dependency Inputs for the large components in the
build.

And if you want faster builds, it would be best to include to
`component=shared_library` too unless you need a fully static link for some
reason.

Note that `incremental_chrome_dll=1` will probably not work on Visual Studio
2008 builds. It may not work on Visual Studio 2010 builds either (pamg couldn't
get it to work as of Nov 2012, encountering numerous link errors). You may have
to use [ninja](ninja_build.md), which has incremental linking on by default.
