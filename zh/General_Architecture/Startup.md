#启动
Chrome大部分时候作为一个独立可执行程序运行，它知道怎样运行我们使用的各种有趣的进程。

这是具体过程：
1. 首先，我们有一个平台相关的入口： Windows上是wWinMain()，Linux上是main()。这存在于chrome/app/chrome_exe_main_* 这样的文件中。在Mac和Windows上，这个函数会加载下面所描述的模块，在Linux上，它会做很少事情，几个平台上，它们最终都会调用：
2. ChromeMain()，它是跨平台代码在所有Chrome进程生命周期中运行的地方，存在于chrome/app/chrome_main*。例如，这是我们调用模块初始化的地方，比如日志和ICU。然后我们会检查其内部 -process-type开关，然后分发给：
3. 一个进程类型相关的主函数，比如BrowserMain()（给外部浏览器进程）或者RenderMain()（给标签页相关的渲染器进程）。


##平台相关入口

###Windows

在Windows上，我们用DLL构建Chrome的组件。wWinMain加载chrome.dll，然后做一些其他随机的事情，然后在DLL中调用ChromeMain。


###Mac

Mac上，Chrome也打包为框架和一个可执行文件，但它们被链接在一块：main()直接调用ChromeMain()。还有第二个入口，在chrome_main_app_mode_mac.mm中，作为app模式快捷方式：“在Mac上，谁也不能用命令行参数创建快捷方式”。相反，我们
Mac is also packaged as a framework and an executable, but they're linked together: main() calls ChromeMain() directly.  There is also a second entry point, in chrome_main_app_mode_mac.mm, for app mode shortcuts: "On Mac, one can't make shortcuts with command-line arguments. Instead, we produce small app bundles which locate the Chromium framework and load it, passing the appropriate data."  This executable also calls ChromeMain().

###Linux

On Linux due to the sandbox we launch subprocesses by repeatedly forking from a helper process.  This means that new subprocesses don't enter through main() again, but instead resume from clones in the middle of startup.  The initial launch of the helper process still executes the normal startup path, so any initialization that happens in ChromeMain() will have been run for all subprocesses but they will all share the same initialization.
评论
您没有权限添加评论。
登录|最近的网站活动|举报滥用行为|打印页面|由 Google 协作平台强力驱动
