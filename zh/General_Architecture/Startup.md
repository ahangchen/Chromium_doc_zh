# 启动
Chrome大部分时候作为一个独立可执行程序运行，它知道怎样运行我们使用的各种有趣的进程。

这是具体过程：
1. 首先，我们有一个平台相关的入口： Windows上是wWinMain()，Linux上是main()。这存在于chrome/app/chrome_exe_main_* 这样的文件中。在Mac和Windows上，这个函数会加载下面所描述的模块，在Linux上，它会做很少事情，几个平台上，它们最终都会调用：
2. ChromeMain()，它是跨平台代码在所有Chrome进程生命周期中运行的地方，存在于chrome/app/chrome_main*。例如，这是我们调用模块初始化的地方，比如日志和ICU。然后我们会检查其内部 -process-type开关，然后分发给：
3. 一个进程类型相关的主函数，比如BrowserMain()（给外部浏览器进程）或者RenderMain()（给标签页相关的渲染器进程）。


## 平台相关入口

### Windows

在Windows上，我们用DLL构建Chrome的组件。wWinMain加载chrome.dll，然后做一些其他随机的事情，然后在DLL中调用ChromeMain。


### Mac

Mac上，Chrome也打包为框架和一个可执行文件，但它们被链接在一块：main()直接调用ChromeMain()。还有第二个入口，在chrome_main_app_mode_mac.mm中，作为app模式快捷方式：“在Mac上，谁也不能用命令行参数创建快捷方式”。相反，我们，做了小的app包，它们找到并加载Chromium框架，传递合适的数据。“这个可执行程序也会调用ChromeMain()。


### Linux

在Linux上，对于沙箱，我们通过重复地从helper进程fork出子进程。这意味着新的子进程不会再次进入main()，但相反的，会在启动过程中，从副本唤醒。helper进程的初始启动仍然在正常的启动路径中执行，所以任何在ChromeMain()中的初始化对于子进程来说都是已经运行过的，但它们都会共享同一个初始化过程。

