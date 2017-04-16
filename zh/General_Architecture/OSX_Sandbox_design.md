# OS X 沙箱设计
这个文档描述了Mac OS X上的进程沙箱机制。

## 背景

沙箱将进程视为一种恶劣的环境，因为进程任何时候都可能被一个恶意攻击者借由缓冲区溢出或者其他这样的攻击方式所影响。一旦进程被影响，我们的目标就变成了，让这个有问题的进程能访问的用户机器的资源越少越好，并尽量避免在标准文件系统访问控制以外，以及内核执行的用户/组进程控制相关的行为。

查看概述文档了解目标与整体架构图表。

## 实现

在Mac OS X上，从Leopard版本开始，每个进程通过使用BSD沙箱设施（在一些Apple的文档中也被成为Seatbelt）拥有自己的权限限制。这由一系列独立的API调用组成，sandbox_init()，设置进程彼时的访问限制。这意味着即使新的权限拒绝访问任何新创建的文件描述符，之前打开的文件描述符仍然生效。我们可以通过在进程启动前正确地设置来利用这一点，在我们将渲染器暴露给任何第三方输入（html，等等）前，切断所有访问。

Seatbelt不会限制内存分配，多线程，或者对先前打开的系统设施的访问。因此，这应该不会影响其他的需求或者严重影响我们的IPC设计。

OS X提供了对缓冲区溢出提供了额外的保护。在Leopard中，栈被标志为不可执行内存，因此更不易被作为执行恶意代码的攻击方向。这不能避免堆的内存溢出，但对于64位应用，除非内存的一部分被显式标识为可执行，否则Leopard不允许任何执行代码的企图。随着我们将来转入64位渲染器进程，这会变成另一个吸引人的安全特性。

sandbox_init()支持预定义沙箱访问限制和提供更精细控制的沙箱配置脚本。

Chromium使用的自定义沙箱配置在源代码树的.sb文件中。

我们定义了下面这些配置文件（路径相对于源代码根目录）：

* content/common/common.sb - 用于所有沙箱的常用安装
* content/renderer/renderer.sb - 用于扩展和渲染器进程。允许访问字体服务器。
* chrome/browser/utility.sb - 用于工具进程。允许访问单一可配置目录。
* content/browser/worker.sb - 用于工作进程。限制度最高 - 除了加载系统库之外，没有文件系统访问权限。
* chrome/browser/nacl_loader.sb - 用户允许不受信任的原生客户端代码（例如，“user”）。

一个让我们不愉快的点是，沙箱进程通过OS X系统API调用。而且没有每个API需要哪些权限的文档，比如它们是否需要访问磁盘文件，或者是否会调用沙箱限制访问的其他API？目前，我们的方法是，在打开沙箱前，对任何可能有问题的API调用做“热身”。例如，颜色配置和共享库可以在我们锁定进程前从磁盘加载。


SandboxInitWrapper::InitializeSandbox()是初始化沙箱的主要入口，它按以下步骤执行：

* 判断当前进程的类型是否需要沙箱化，如果需要，判断需要使用哪种沙箱配置。
* 通过调用sandbox::SandboxWarmup() “热身”相关"系统API。
* 通过调用sandbox::EnableSandbox()启动沙箱。

### 调试

OS X沙箱实现支持下面这些标志位：

* --no-sandbox - 关闭整个沙箱
* --enable-sandbox-logging - 关于哪个系统调用正在阻塞的详细信息被记录到syslog

[Debugging Chrome on OS X](https://www.chromium.org/developers/how-tos/debugging-on-os-x)里有更多关于调试和Mac OS X 沙箱API诊断工具的文档。

## 扩展阅读

* http://reverse.put.as/wp-content/uploads/2011/09/Apple-Sandbox-Guide-v1.0.pdf
* http://www.318.com/techjournal/?p=107
* 沙箱手册页 (man 7 sandbox)
* 系统沙箱文件可以在下面的路径之一找到(取决于系统版本)：

  * /Library/Sandbox/Profiles
  * /System/Library/Sandbox/Profiles
  * /usr/share/sandbox