#OS X 沙箱设计
这个文档描述了Mac OS X上的进程沙箱机制。

##背景

沙箱将进程视为一种恶劣的环境，因为进程任何时候都可能被一个恶意攻击者借由缓冲区溢出或者其他这样的攻击方式所影响。一旦进程被影响，我们的目标就变成了，让这个有问题的进程能访问的用户机器的资源越少越好，并尽量避免在标准文件系统访问控制以外，以及内核执行的用户/组进程控制相关的行为。

查看概述文档了解目标与整体架构图表。

##实现

在Mac OS X上，从Leopard版本开始，每个进程通过使用BSD沙箱设施（在一些Apple的文档中也被成为Seatbelt）拥有自己的权限限制。这由一系列独立的API调用组成，sandbox_init()，设置进程彼时的访问限制。这意味着即使新的权限拒绝访问任何新创建的文件描述符，之前打开的文件描述符仍然生效。我们可以通过在进程启动前正确地设置来利用这一点，在我们将渲染器暴露给任何第三方输入（html，等等）前，切断所有访问。


Seatbelt does not place restrictions on memory allocation, threading, or access to previously opened OS facilities. As a result, this shouldn't impose any additional requirements or drastically alter our IPC designs. 

OS X provides additional protection against buffer overflows. In Leopard, the stack is marked as non-executable memory and thus less susceptible as an attack vector for executing malicious code. This doesn't prevent against buffer overruns in the heap, but for 64-bit apps, Leopard disallows any attempts to execute code unless that portion of memory is explicitly marked as executable. As we move towards 64-bit render processes in the future, this will be another attractive security feature. 

sandbox_init() has supports for both predefined sandbox access restrictions and sandbox profile scripts which provide finer grained control.

Chromium uses custom sandbox profiles defined in .sb files in the source tree.

The following profiles are defined (paths relative to root of source directory):
* content/common/common.sb - used for common setup for all sandboxes.
* content/renderer/renderer.sb - used for the extension & renderer processes. Enables access to the font server.
* chrome/browser/utility.sb - used by the utility process. Allows access to a single configurable directory.
* content/browser/worker.sb - used by the worker process.  Most restrictive - no file system access apart from loading system libraries.
* chrome/browser/nacl_loader.sb - used for running Native Client untrusted (i.e., "user") code.

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