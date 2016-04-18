#OS X 沙箱设计
这个文档描述了Mac OS X上的进程沙箱机制。

##背景

沙箱将进程视为一种恶劣的环境，因为进程任何时候都可能被一个恶意攻击者借由缓冲区溢出或者其他这样的攻击方式所影响。一旦进程被影响，我们的目标就变成了，让这个有问题的进程能访问的用户机器的资源越少越好，并尽量避免在标准文件系统访问控制以外，以及内核执行的用户/组进程控制相关的行为。

查看概述文档了解目标与整体架构图表。

##实现

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

One sticky point we run into is that the sandboxed process calls through to OS X system APIs. There is no documentation available about which privileges each API needs, such as whether they need access to on-disk files, or call other APIs to which the sandbox restricts access. Our approach to date has been to "warm up" any problematic API calls before turning the sandbox on. This means that we call through to the API, to allow it to cache whatever resource it needs. For example, color profiles and shared libraries can be loaded from disk before we "lock down" the process.

SandboxInitWrapper::InitializeSandbox() is the main entry point for initializing the Sandbox, it performs the following steps:
* Determines if the current process type needs to be sandboxed and if so, which sandbox configuration to use.
* "Warm up" relevant system APIs by calling through to  sandbox::SandboxWarmup() .
* Enable the Sandbox by calling through to  sandbox::EnableSandbox() .

###Diagnostics

The OS X sandbox implementation supports the following flags:
* --no-sandbox - Disable the sandbox entirely.
* --enable-sandbox-logging - Verbose information about which system calls are blocked is logged to syslog.

[Debugging Chrome on OS X](https://www.chromium.org/developers/how-tos/debugging-on-os-x) contains more documentation on debugging and diagnostic tools provided by the Mac OS X sandbox API.

##Additional Reading

* http://reverse.put.as/wp-content/uploads/2011/09/Apple-Sandbox-Guide-v1.0.pdf
* http://www.318.com/techjournal/?p=107
* sandbox man page (man 7 sandbox)
* System sandbox files can be found under one of the following paths (depending on the OS Version):

  * /Library/Sandbox/Profiles
  * /System/Library/Sandbox/Profiles
  * /usr/share/sandbox