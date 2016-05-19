#Android上的Java资源
##概览

Android上的Chrome使用了Java代码中的一些资源（例如，Android layout和相关的字符串或图像）。这些资源依照Android的资源目录结构存储于Java根目录内，

* content/public/android/java/res - Java resources available within content and anything that depends on content
* chrome/android/java/res - Java resources available within chrome and anything that depends on chrome
* ui/android/java/res - Java resources available within ui and anything that depends on ui

Java代码可以使用自动生成的R类正常地引用这些资源，但请确认使用正确的包名修饰R类。
```java
// Use a resource from content
setImageResource(org.chromium.content.R.drawable.globe_favicon);

// Use a resource from chrome
setContentView(org.chromium.chrome.R.layout.month_picker);
```
##资源如何打包

在编译Java代码时，会基于Java上下文资源生成一个R.java文件。这个R.java包含了一些非常量的变量，只在编译（以及任何基于content的非APK目的）的时候使用，R.java不会被包含在content jar包中。

在构建一个APK的时候，比如content_shell_apk，资源会从content、任何其他依赖、content shell本身中合并进来。这些合并的资源会得到处理并包含在APK中。基于这些资源，会生成一个有着正确的资源-&gt;ID映射的新的R.java文件。这个R.java会被复制到每个依赖R的包中 (例如. org.chromium.content.R和org.chromium.content_shell.R), 所有这些副本会被包含在APK中。

This process closely follows Android's handling of resources in library projects, where content and chrome are the "libraries", though we don't use the SDK to compile our "libraries".  Hence some of the same caveats apply.  In particular, two resources with the same ID cannot coexist.  The resource highest in the dependency chain (e.g. in content shell) will override the others (e.g. in content).
##Supporting resources in gyp

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