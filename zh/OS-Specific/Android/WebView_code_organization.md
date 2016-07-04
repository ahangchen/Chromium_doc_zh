# Android WebView代码组织

*android-webview-dev@chromium.org*

*Last updated Jul 2014.*

![](webview_org.png)

Android build tree中，external/chromium_org是chromium源代码(根目录位于[src.chromium.org/viewvc/chrome/trunk/src/](https://src.chromium.org/viewvc/chrome/trunk/src/))的一个镜像。


[CHROMIUM TREE](https://src.chromium.org/viewvc/chrome/trunk/src/) (“upstream”: trunk/src; “downstream”: external/chromium_org)
- android_webview/java
  - Webview chromium栈的顶层入口点
  - 在chromium代码栈上为downstream android代码树的使用提供一个半稳定的表层/wrapper。不像其他chromium java代码，这个包中public类的public接口被作为第一类public API,而使用方则与这个git仓库版本独立。
  - 对于大部分WebView API，后端功能由chromium [content module](http://dev.chromium.org/developers/content-module), 或者辅助[browser components](http://dev.chromium.org/developers/design-documents/browser-components)提供；而对于它分发的feature，则在chromium栈的其他层调用public Java API，再由android webview/native 目录中的代码调用native JNI的部分。

- android_webview/native
  - Would be more appropriately named ‘jni’. In general code in here is responsible for crossing the Java<->native boundary. Classes here tend to reflect their java counterpart naming, and primarily responsible for creational and object ownership and cleanup roles
  - Where possible avoid placing complex implementation logic in here instead factor out into more focused feature specific classes inside the android_webview/browser folder or as a component in the top-level components/ folder (e.g. if needs to be shared with Chrome browser).
  - By convention code in here should follow the same threading model as the public WebView API (that is, primarily executes on UI thread). Non-trivial interaction with BrowserThreads and IPC to renderer is extracted down to the browser/ folder.
- android_webview/browser
  - Provides non-trivial behavioral and functional classes for implementing features not directly exported by the content module public API.
  - By convention, any complex native threading code should be encapsulated here.
  - DEPS enforces that classes in this folder have no static dependency on the higher layer android_webview/native JNI wrappers. This aims to make the pieces in browser/ more modular and testable, and minimize the [big ball of mud tendency](http://laputan.org/mud/mud.html).
- android_webview/renderer
  - Contains all code that logically resides in the renderer process. While WebView currently only supports single process mode, the browser/renderer separation is maintained as it is a useful architectural separation between the (implicitly trusted) java application and the (potentially untrusted) web-platform code.
- android_webview/lib
  - This is the main entry point to the libwebviewchromium.so native library. This is the top of the native code static dependency tree; no other module may depend on this.
- android_webview/common
  - Declares raw types etc that are shared by both android_webview/browser and android_webview/renderer
  - While single process mode makes it possible to share state via e.g. globals hidden in this module, resist this temptation. Instead common abstract types may be declared here, but the concrete instances should be constructed and passed in from the appropriate creational class in android_webview/lib.
  - As per chromium conventions, IPC message types are declared here.
- android_webview/public
  - Defines and exports abstract native interfaces for certain performance critical (e.g. rendering pipeline) functionality and that cannot be implemented via Java.
  - This resolves the conflicting needs: of keeping all chromium code SDK/NDK clean, yet using certain internal Android platform facilities to implement a backwards compatible and high-performance framework custom View class.
  - To minimize ABI interdependencies (e.g. avoid STL types being passed over .so boundary, incompatible new/delete calls across boundaries, etc) the interfaces declared here use simple C-style calling conventions (POD structs and static methods, injected via tables of simple function pointer).
  - The plat_support module in frameworks/webview provides the canonical concrete realization of the abstract interfaces declared here.
- android_webview/javatests
  - Integration tests for the org.chromium.android_webview.\* APIs. 
    - These tests exercise the top-level APIs used by the Android ‘glue layer’ to actually implement the WebView API, so they don’t actually test the full WebView stack,
    - They do however have access to internal APIs and state that is unavailable further up the stack.
    - Integration tests extend AwTestBase.
  - Unit tests for functionality implemented entirely in Java (for example AwLayoutSizerTest.java). Unit tests extend InstrumentationTestCase.
- android_webview/unittestjava
  - Java support code for C++ unittests. This is necessary to test any code that uses JNI, for example any logic checked into the android_webview/native layer.


[ANDROID TREE](https://android.googlesource.com/)
- [frameworks/base](https://android.googlesource.com/) -- core/java/android/webkit/…
  - Defines the public API to the android.webkit package (WebView, WebSettings, etc)
  - Declares the [hidden] abstract WebViewFactoryProvider, WebViewProvider interface that webview implementations must subclass.
  - Defines various “POD” like data types that are passed between the application and the concrete WebView implementation (e.g. WebViewTransport) and ancillary utility classes (e.g. URLUtil)
  - (Historic; pre-KK) Defines the WebViewClassic & related classes that powered the legacy webview implementation
- [frameworks/webview](https://android.googlesource.com/platform/frameworks/webview/) -- chromium/java
  - often referred to as the “glue layer” this bridges between the core android framework, and external/chromium_org
  - Performs dependency inversion so frameworks/base and external/chromium_org have no interdependency on internal details of each other. This is the main entry-point on the java side for the chromium WebViewFactory
  - The goal is this should only depend on android_webview public APIs (some accidental exceptions exist, e.g. ThreadUtils, LibraryLoader etc) and not contain complex logic; just method forwarding (interface adaption) across the boundary.
  - By convention, this module also provides all the mappings from embedding application targetSdkVersion to specific set of settings for workarounds & quirks that should be runtime enabled in the underlying chromium stack
- [frameworks/webview](https://android.googlesource.com/platform/frameworks/webview/) --   - chromium/plat_support
  - provides a native support library to bind a select few internal native platform APIs to android_webview/public counterparts (e.g. GL functor, gralloc, and skia bitmap access utilities).
  - Dependency injection of code in this folder avoids any build-time dependency of external/chromium_org on non-NDK symbols.
- [cts](https://android.googlesource.com/platform/cts/) -- tests/tests/webkit/src/android/webkit/cts
  - Android Compatibility Test Suite for the android.webkit.\* APIs. These tests have only access to the ‘public’ WebView API and test the WebView implementation in an configuration identical to when it’s actually running in an application.
  - There should be at least one test for every publicly exposed API in the suite. This makes sure everything is ‘hooked up’ correctly.
