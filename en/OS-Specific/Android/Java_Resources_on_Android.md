# Java Resources on Android
## Overview

Chrome for Android uses certain resources in Java code (e.g. Android layouts and associated strings or images).  These resources are stored according to Android's resource directory structure within a Java root folder.
* content/public/android/java/res - Java resources available within content and anything that depends on content
* chrome/android/java/res - Java resources available within chrome and anything that depends on chrome
* ui/android/java/res - Java resources available within ui and anything that depends on ui

Java code can reference these resources in the normal way using a generated R class, being sure to qualify it with the correct package name.
```java
// Use a resource from content
setImageResource(org.chromium.content.R.drawable.globe_favicon);

// Use a resource from chrome
setContentView(org.chromium.chrome.R.layout.month_picker);
```
## How resources are packaged

While compiling the Java code in content, an R.java file is generated based on the Java resources in content.  This R.java contains non-final constants and is used only while compiling content (and any non-APK target that depends on content) but is not included in the content jar.

When building an APK target, such as content_shell_apk, resources are merged from content, any other dependencies, and from content shell itself.  These merged resources are processed and included in the APK.  Based on these resources, a new R.java is generated with the correct resource -> ID mappings.  This R.java is copied into the R packages needed by each dependency (e.g. org.chromium.content.R and org.chromium.content_shell.R), and all these copies are included in the APK.

This process closely follows Android's handling of resources in library projects, where content and chrome are the "libraries", though we don't use the SDK to compile our "libraries".  Hence some of the same caveats apply.  In particular, two resources with the same ID cannot coexist.  The resource highest in the dependency chain (e.g. in content shell) will override the others (e.g. in content).
## Supporting resources in gyp

To add resources to another Java root folder, add the variables has_java_resources, R_package, and R_package_relpath to the gyp target that builds that Java code.  For example:
```json
{
  'target_name': 'content_java',
  'type': 'none',
  'dependencies': [ ... ],
  'variables': {
    'package_name': 'content',
    'java_in_dir': '../content/public/android/java',

    # Support Java resources in content
    'has_java_resources': 1,
    'R_package': 'org.chromium.content',
    'R_package_relpath': 'org/chromium/content',
  },
  'includes': [ '../build/java.gypi' ],
},
```