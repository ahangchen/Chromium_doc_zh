#沙箱FAQ
###什么是沙箱？

沙箱是一个允许沙箱进程创建的C++库，沙箱进程是一种运行在非常限制性的环境中的进程。沙箱进程可以唯一自由使用的资源是CPU周期和内存。例如，沙箱进程不能写磁盘或者显示他们自己的窗口。它们真正能做的事情由一种明确的策略锁控制。Chromium渲染器都是沙箱化进程。


###沙箱可以保护什么，不能保护什么？

沙箱限制了运行在沙箱中的代码的bug的危害。这些bug不能在用户的账号中安装持久性的恶意软件（因为写文件系统被禁止），这些bug也不能读取或者从用户的设备中盗取任何文件。

（在Chromium中，渲染器进程是沙箱化的，它们处于这种保护中。Chromium插件还没有运行在沙箱中，因为许多插件的设计基于这样一个假设：它们对本地系统有着完全的访问权限。另外也要注意，Chromium渲染器进程与系统相隔离，但还未与网络相隔离。所以，基于域名的数据隔离还未提供）。

沙箱不能为系统组件（比如系统内核正在运行的组件）中的bug提供任何保护。


###沙箱像JVM？

恩，有点像...除了你必须为Java沙箱的优点重写代码以使用Java。在我们的沙箱中，你可以向你现有的C/C++应用程序添加沙箱。由于代码并非执行于虚拟机中，你可以得到原生的速度，以及对Windows API的直接访问。



###Do I need to install a driver or kernel module? Does the user need to be Administrator?

No and no. The sandbox is a pure user-mode library, and any user can run sandboxed processes.

###How can you do this for C++ code if there is no virtual machine?

We leverage the Windows security model. In Windows, code cannot perform any form of I/O (be it disk, keyboard, or screen) without making a system call. In most system calls, Windows performs some sort of security check. The sandbox sets things up so that these security checks fail for the kinds of actions that you don’t want the sandboxed process to perform. In Chromium, the sandbox is such that all access checks should fail.

###So how can a sandboxed process such as a renderer accomplish anything?

Certain communication channels are explicitly open for the sandboxed processes; the processes can write and read from these channels. A more privileged process can use these channels to do certain actions on behalf of the sandboxed process. In Chromium, the privileged process is usually the browser process.

###Doesn't Vista have similar functionality? 

Yes. It's called integrity levels (ILs). The sandbox detects Vista and uses integrity levels, as well. The main difference is that the sandbox also works well under Windows XP. The only application that we are aware of that uses ILs is Internet Explorer 7. In other words, leveraging the new Vista security features is one of the things that the sandbox library does for you.

###This is very neat. Can I use the sandbox in my own programs?

Yes. The sandbox does not have any hard dependencies on the Chromium browser and was designed to be used with other Internet-facing applications. The main hurdle is that you have to split your application into at least two interacting processes. One of the processes is privileged and does I/O and interacts with the user; the other is not privileged at all and does untrusted data processing.

###Isn't that a lot of work?

Possibly. But it's worth the trouble, especially if your application processes arbitrary untrusted data. Any buffer overflow or format decoding flaw that your code might have won't automatically result in malicious code compromising the whole computer. The sandbox is not a security silver bullet, but it is a strong last defense against nasty exploits.

###Should I be aware of any gotchas?

Well, the main thing to keep in mind is that you should only sandbox code that you fully control or that you fully understand. Sandboxing third-party code can be very difficult. For example, you might not be aware of third-party code's need to create temporary files or display warning dialogs; these operations will not succeed unless you explicitly allow them. Furthermore, third-party components could get updated on the end-user machine with new behaviors that you did not anticipate.

###How about COM, Winsock, or DirectX — can I use them?

For the most part, no. We recommend against using them before lock-down. Once a sandboxed process is locked down, use of Winsock, COM, or DirectX will either malfunction or outright fail.

###What do you mean by before lock-down? Is the sandboxed process not locked down from the start?

No, the sandboxed process does not start fully secured. The sandbox takes effect once the process makes a call to the sandbox method LowerToken(). This allows for a period during process startup when the sandboxed process can freely get hold of critical resources, load libraries, or read configuration files. The process should call LowerToken() as soon as feasible and certainly before it starts interacting with untrusted data. 

Note: Any OS handle that is left open after calling LowerToken() can be abused by malware if your process gets infected. That's why we discourage calling COM or other heavyweight APIs; they leave some open handles around for efficiency in later calls.

###So what APIs can you call?

There is no master list of safe APIs. In general, structure your code such that the sandboxed code reads and writes from pipes or shared memory and just does operations over this data. In the case of Chromium, the entire WebKit code runs this way, and the output is mostly a rendered bitmap of the web pages. You can use Chromium as inspiration for your own memory- or pipe-based IPC.

###But can't malware just infect the process at the other end of the pipe or shared memory?

Yes, it might, if there's a bug there. The key point is that it's easier to write and analyze a correct IPC mechanism than, say, a web browser engine. Strive to make the IPC code as simple as possible, and have it reviewed by others.