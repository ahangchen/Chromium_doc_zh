# Log #

[TOC]


## 概要

Log曾经由Android的[android.util.Log](http://developer.android.com/reference/android/util/Log.html)来实现。

现在有一个新的wrapper可以用：org.chromium.base.Log。它设计用于在单一的类之外，作为同一个逻辑族来写日志，并且使得为不同的族打开和关闭日志变得容易。

用法:

```java
private static final String TAG = "YourModuleTag";
...
Log.i(TAG, "Logged INFO message.");
Log.d(TAG, "Some DEBUG info: %s", data);
```

Output:

```
I/cr_YourModuleTag: ( 999): Logged INFO message
D/cr_YourModuleTag: ( 999): [MyClass.java:42] Some DEBUG info: data.toString
```

在这里，**TAG**可以是一个特性或者package的名字，比如“MediaRemote”或“NFC”。在大多数情况下，类名是不需要的。chromium会预先考虑使用"cr_"前缀来标记来自chrome的那些log。

### Verbose和Debug类型的log有特殊的处理 ###

*   `org.chromium.base.Log`对`Log.v`和`Log.d`的调用会被Proguard从二进制的程序中剥离出来，在release版本中，没有办法可以获得这些log。

*   文件名和行号不会出现在log信息中。
    对于高优先级的日志，处于性能考虑，它们也不会被添加。

### 当异常作为最后的参数时，打印异常堆栈 ###

像`java.util.Log`，那样，把一个throwable对象作为最后一个参数，可以dump出对应的堆栈：

```java
Log.i(TAG, "An error happened: %s", e)
```

```
I/cr_YourModuleTag: ( 999): An error happened: This is the exception's message
I/cr_YourModuleTag: ( 999): java.lang.Exception: This is the exception's message
I/cr_YourModuleTag: ( 999):     at foo.bar.MyClass.test(MyClass.java:42)
I/cr_YourModuleTag: ( 999):     ...
```

把exception作为最后的参数不会妨碍它用于string格式化。

## Log最佳实践

### Rule #1: Never log PII (Personal Identification Information):

这是一个大的担忧，因为其他程序可以访问日志并通过这种方式提取大量的数据。即使JellyBean下限制了这种新闻，人们也可以在你的root设备上运行你的程序并允许一些app去访问它们。另外，任何有设备USB权限的人可以用adb获得完整的logcat，并且立即获得相同的数据。

如果你真的需要记录一些东西，相反地，你可以打一串X(e.g."XXXXXXX")，或者打印PII的truncated hash。Truncation是需要的，这能让攻击者更难通过彩虹表或者相似的手段恢复完整的数据。

类似的，避免dump API key， cookies，等等……

### Rule #2: 不要在产品代码里打debug日志:

在release构建中，用Proguard移除日志方法。因为日志信息不可写，所以创建它们的代价需要避免。这可以用三种补充方式来完成：

#### 使用string formatting而非concate

```java
// BAD
Log.d(TAG, "I " + preference + " writing logs.");

// BETTER
Log.d(TAG, "I %s writing logs.", preference);
```
Progurad删除了Log的方法，但不会对参数做任何事情，方法参数仍然会被计算并作为输入提供。上面的第一种调用总是会导致`StringBuilder`的创建以及一些连接，而第二种方法只是传递参数，不需要做那样的事情。

#### 排除昂贵的调用

有时候log的内容并非确实可用，需要特殊的计算，当日志被关掉时，这应当避免。

```java
static private final boolean DEBUG = false;  // debug toggle.
...
if (DEBUG) {
  Log.i(TAG, createThatExpensiveLogMessage(activity))
}
```

因为变量DEBUG是`static final`的，需要在编译时计算，java编译器会在生成的`.class`文件中，优化掉所有被debug排除的调用。然而这种修改需要编辑每个需要debug的文件并且重新编译。

#### 为debug函数使用`@RemovableInRelease`注解

这个注解告诉Proguard给定的这个函数没有边缘效应，它的调用只是为了它的返回值。如果这个值没有被使用，这个调用可用被移除。如果函数根本没有被调用，它也会被移除。因为Proguard已经习惯于对debug构建剥离debug和verbose调用，这个注解允许Proguard做更深入的操作，即，移除生成log调用参数的生成方法。

```java
/* If that function is only used in Log.d calls, proguard should
 * completely remove it from the release builds. */
@RemovableInRelease
private static String getSomeDebugLogString(Thing[] things) {
  StringBuilder sb = new StringBuilder(
      "Reporting " + thing.length + " things: ");
  for (Thing thing : things) {
    sb.append('\n').append(thing.id).append(' ').append(report.foo);
  }
  return sb.toString();
}

public void bar() {
  ...
  Log.d(TAG, getSomeDebugLogString(things)); /* The line is removed in
                                              *  release builds. */
}
```

我们再次看到，只有在函数的输入是局域中已经可用的变量时，这才是有用的。这个想法是把计算，连接，等等操作放到一个不需要用到时无法移除的地方，而不影响主方法的逻辑。这跟监控一个在debug中可用在release中不可用的static final属性有相似的效果。

### Rule #3: 多用短日志

这与全局的用于保存所有日志的固定长度内核buffer有关。试着让你的日志信息尽可能简洁。这能在一些淘气的事情发生时，减少他人感兴趣的日志数据溢出缓冲区的风险。使用单行日志比多行日志更好，比如，不要使用：

```java
Log.GROUP.d(TAG, "field1 = %s", value1);
Log.GROUP.d(TAG, "field2 = %s", value2);
Log.GROUP.d(TAG, "field3 = %s", value3);
```

相反地，这样些：

```java
Log.d(TAG, "field1 = %s, field2 = %s, field3 = %s", value1, value2, value3);
```

如果你数字符个数，似乎没什么不同，但每个独立日志入口也在内核log buffer中实现了一个小的，但不可忽视的header。因为每个字节都会占用空间，你也能试一些短的东西，比如：

```java
Log.d(TAG, "fields [%s,%s,%s]", value1, value2, value3);
```

## 过滤日志

Logcat允许通过指定tag和相关的level来执行过滤

```shell
adb logcat [TAG_EXPR:LEVEL]...
adb logcat cr_YourModuleTag:D *:S
```

这些命令只会为`cr_YourModuleTag`输出与DEBUG等级相同或者比DEBUG等级高的日志，并让其他任何东西保持缄默（这个等级或更高等级上，没有东西被输出，所以说它使其他tag缄默了）。你可以通过设置一个环境变量来一直使用一个过滤器：

```shell
export ANDROID_LOG_TAGS="cr_YourModuleTag:D *:S"
```

这种语法不支持tag泛化或者正则表达式（除了`*`，代表所有tag）。如果要进一步提炼你的过滤器，使用`grep`或者类似的工具。

更多内容，可以参考[related page on developer.android.com](http://developer.android.com/tools/debugging/debugging-log.html#filteringOutput)

## JUnit测试中的Log

我们使用[robolectric](http://robolectric.org)来运行JUnit测试。它用“Shadow”类代替了一些android framework类，以确保我们可以在标准的JVM运行我们的代码，默认情况下，调用`Log`方法不会打印任何东西。

这种默认设置在通常的设置中是不能修改的，但如果你需要在局部启用日志或者进行一种特殊的测试，只要添加下面几行代码到你的测试中：

```java
@Before
public void setUp() {
  ShadowLog.stream = System.out;
  //you other setup here
}
```
