# JNI on Chromium for Android
文档和快速引用可以在sample_for_tests.h和SampleForTests.java找到。

## Overview

On other platforms, the underlying system / platform APIs (win32, cocoa, gtk) are directly accessible via C/C++.

On Android, the underlying userland API requires JNI (Java Native Interface) to bridge the native C/C++ code. 

Essentially, JNI is a glorified dlsym / GetProcAddress to allow linking / resetting function pointers by name.

Chromium for Android uses JNI in basically two ways:

1) To implement an existing chromium cross-platform C++ API in android.
For instance, base::PathProvider implementation on android requires a "system call", in java.
In this case, we have a thin wrapper / adapter in java that exposes the api to be called by native code. The current structure:
base/android/path_utils.(h|cc)
base/android/java/PathUtils.java

that is, the .java and .cc files are located close to each other.

2) To bind the higher level components with the rest of chromium.
The higher level components are in Java and goes without saying that they need to call into native code.

As you can imagine, there's a considerable amount of boilerplate code to get all these bindings in place. Plus, JNI itself is not type safe, and relies heavily on varargs.

In order to automate this and reduce bugs we had when we were writing our JNI ourselves, we wrote a tool called “jni binding generator”. It’s currently generating ~57 files, ~9000 lines.

This tool generates the two-way (i.e, both native => java and java => native) stubs / wrappers / bindings:
1) From java code, a method that is declared as “native int getMeFooBar(String zoo)” needs to be registered by native code, that is, the native side needs to set its function pointer to an appropriate C function.

2) The java side may expose a method to the native side. Such methods are annotated with “@CalledByNative”, and the generator will create the corresponding bindings. In order to effectively bridge C=>C++, we also have a few conventions on C++ object pointers. For instance, the first parameter may represent a C++ object, and then the bindings will automatically generate the appropriate cast and call into C++ code (JNI itself is only about C).

The tool has been extensively used in chromium for android, and has comprehensive set of tests.
On top of regular python unit tests, it also has a self-documenting “SampleForTests.java” that it uses to generate the bindings code, a corresponding .cc file that includes the generated code, and a target for building the file and ensure the generator is creating valid code.

The tool itself is located under:
base/android/jni_generator/

The reason for it to be in base is that the generated code and its tests depend on base.

## GYP Rules for jni headers

There are two gypi files that provides rules for generating jni bindings for Java-files.
The first one sits in //build/jni_generator.gypi
To use this, create a gyp target with the following form:
```
 {
   'target_name': 'base_jni_headers',
   'type': 'none',
   'sources': [
     'android/java/src/org/chromium/base/BuildInfo.java',
     ...
     'android/java/src/org/chromium/base/SystemMessageHandler.java',
   ],
   'variables': {
     'jni_gen_dir': 'base',
   },
   'includes': [ '../build/jni_generator.gypi' ],
 },
```
The second file is //build/system_classes_jni_generator.gypi and is meant to be used with system Java-files such as java/io/InputStream.class
To use this, create a gyp target with the following form:
```
 {
   'target_name': 'chrome_jni_headers_for_system_classes',
   'type': 'none',
   'sources': [
     'android/java/src/org/chromium/chrome/browser/AndroidProtocolAdapter.java',
     ...
   ],
   'variables': {
     'jni_gen_dir': 'chrome',
     'input_java_class' : 'java/io/InputStream.class',
   },
   'includes': [ '../build/system_classes_jni_generator.gypi' ],
 },
```
Updating the Autogenerator

On https://src.chromium.org/viewvc/chrome/trunk/src/base/android/jni_generator/jni_generator.gyp?view=markup , there's a test target that will run the python tests for the generator, and also generate a header from a sample Java app and finally compile with a sample cc. If you change the generator, please add a test case in jni_macro_generator_tests.py / SampleForTests.java / sample_for_tests.cc: