#Sandbox
安全是Chromium最重要的目标之一。安全的关键在于理解下面这点：在我们完整地理解了系统在所有可能的输入组合下表现出的行为之后，我们才能够真的保证系统安全。对于像Chromium这样庞大而多样化的代码库，推理它的各个部分可能的行为的组合几乎是不可能的。沙箱的目标是提供这样一种保证：不论输入什么，保证一段代码最终能或不能做的事情。

沙盒利用操作系统提供的安全性，允许不能对计算机做出持久性改变或者访问持续变化的信息的代码的执行。沙箱提供的架构和具体保证依赖于操作系统。这个文档覆盖了Windows实现与一般的设计。Linux实现和OSX实现也会在这里描述。

如果你不想要阅读这整个文档，你可以阅读[Sandbox FAQ](Sandbox_FAQ.md)。沙箱保护与不保护的内容也可以在FAQ中找到。


##设计原则

* **不要重新发明轮子**: 用更好的安全模型扩展操作系统内核很有诱惑力。但不要这样做。让操作系统在所控制的对象上应用它的安全策略。另一方面，创建有自定义安全模型的应用程序层级对象（抽象）是可以的。
* **最小权限原则**: 这既应该用于沙箱代码也应该用于控制沙箱的代码。换言之，即使用于不能提升权限到超级用户，沙箱也需要能够工作。
* **假定沙盒代码是恶意代码**: 出于威胁建模的目的，我们认为沙箱中的代码一旦执行路径越过了一些main()函数的早期调用，那么它是有害的（即，它会运行有害代码），实践中，在第一外部输入被接收时，或者在进入主循环前，这就可能发生。
* **敏感**: 非恶意代码不会尝试访问它不能获得的资源。在这种情况下，沙箱产生的性能影响应该接近零。一旦敏感资源需要以一种控制行为访问时，一点性能损失是必要的。这是在操作系统安全合适事情情况下的常见例子。
* **仿真不是安全**: 仿真和虚拟机方案本身不能提供安全。沙箱不会出于安全目的，依赖于代码仿真，或者代码转换，或者代码修复。
##沙箱windows架构

Windows沙箱是一种仅用户模式可用的沙箱。没有特殊的内核模式驱动，用户不需要为了沙箱正确运行而成为管理员。沙箱设计了32位和64位两种进程，在所有windows7和windows10之间的所有操作系统版本都被测试过。

沙箱在进程级粒度进行运作。凡是需要沙箱化的任何东西需要放到独立进程里运行。最小化沙箱配置有两个过程：一个是被成为broker的权限控制器，以及被称为target的一个或多个沙箱化进程。在整个文档和代码中这两个词有着上述两种精确的内涵。沙箱是一个必须被链接到broker和target可执行程序的静态库。

###broker进程

在Chromium中，broker总是浏览进程。broker，广泛概念里，是一个权限控制器，沙箱进程活动的管理员。broker进程的责任是：

1. 指定每个目标进程中的策略
2. 生成目标进程
3. 维护沙箱策略引擎服务
4. 维护沙箱拦截管理器
5. 维护沙箱IPC服务（与target进程的通信）
6. 代表目标进程执行策略允许的操作。

broker应该始终比所有它生成的目标进程还要活的久。沙箱IPC是一种低级别的机制（与ChromiumIPC机制不同），这些调用会被策略评估。策略允许的调用会由broker执行，结果会通过同样的IPC返回给目标进程。拦截管理器是为应该通过IPC转发给broker的windows API调用提供补丁。

###目标进程
在Chromium中，渲染器总是target进程，除非浏览进程被指定了--no-sandbox命令行参数。target进程维护所有将在沙箱中允许的代码，以及沙箱基础设施的客户端：

1. 所有代码沙箱化
2. 沙箱IPC客户端
3. 沙箱策略引擎客户端
4. 沙箱拦截

第2,3,4条是沙箱库的一部分，与需要被沙箱化的代码关联。

拦截器（也称为hook）是通过沙箱转发的Windows API调用。由broker重新发出API 调用，并返回结果或者干脆终止调用。拦截器+IPC机制不能提供安全性；它的目的是在沙箱中的代码因沙箱限制不能修改时，提供兼容性。为了节省不必要的IPC，在进行IPC调用前，target中进程策略也会被评估，尽管这不是用作安全保障，但这仅仅是一个速度优化。

期望在未来大部分plugin会运行在target进程里。


![](sbox_top_diagram.PNG)

##沙箱限制

在它的核心，沙箱依赖于4个Windows提供的机制：

* 限定的token
* Windows工作对象
* Windows桌面对象
* Windows Vista及以上:集成层

这些机制在保护操作系统，操作系统的限制，用户提供的数据上相当的高效，前提是：

* 所有可以安全化的资源都有一个比null更好的安全描述符。换言之，没有关键资源会有错误的安全配置。
* 计算机并未被恶意软件所损害。
* 第三方软件不能弱化系统安全。

** 注意：上面具体的措施以及在内核外的措施会在下面的“进程措施”部分阐述。

###Token

One issue that other similar sandbox projects face is how restricted can the token and job be while still having a properly functioning process. For the Chromium sandbox, the most restrictive token for Windows XP takes the following form:

**Regular Groups**

Logon SID : mandatory

All other SIDs : deny only, mandatory

**Restricted Groups**

S-1-0-0 : mandatory

**Privileges**

None

With the caveats described above, it is near impossible to find an existing resource that the OS will grant access with such a token. As long as the disk root directories have non-null security, even files with null security cannot be accessed. In Vista, the most restrictive token is the same but it also includes the low integrity level label. The Chromium renderer normally runs with this token, which means that almost all resources that the renderer process uses have been acquired by the Browser and their handles duplicated into the renderer process.

Note that the token is not derived from anonymous or from the guest token; it is derived from the user's token and thus associated to the user logon. As a result, any auditing that the system or the domain has in place can still be used.

By design, the sandbox token cannot protect the following non-securable resources:
* Mounted FAT or FAT32 volumes: The security descriptor on them is effectively null. Malware running in the target can read and write to these volumes as long it can guess or deduce their paths.
* TCP/IP: The security of TCP/IP sockets in Windows 2000 and Windows XP (but not in Vista) is effectively null. It might be possible for malicious code in the target to send and receive network packets to any host.
* 
More information about the Windows token object can be found at [02].

###The Job object

The target process also runs under a Job object. Using this Windows mechanism, some interesting global restrictions that do not have a traditional object or security descriptor associated with them are enforced:
* Forbid per-use system-wide changes using SystemParametersInfo(), which can be used to swap the mouse buttons or set the screen saver timeout
* Forbid the creation or switch of Desktops
* Forbid changes to the per-user display configuration such as resolution and primary display
* No read or write to the clipboard
* Forbid Windows message broadcasts
* Forbid setting global Windows hooks (using SetWindowsHookEx())
* Forbid access to the global atoms table
* Forbid access to USER handles created outside the Job object
* One active process limit (disallows creating child processes)

Chromium renderers normally run with all these restrictions active. Each renderers run in its own Job object. Using the Job object, the sandbox can (but currently does not) prevent:
* Excessive use of CPU cycles
* Excessive use of memory
* Excessive use of IO

More information about Windows Job Objects can be found in [1]. 

###The alternate desktop

The token and the job object define a security boundary: that is, all processes with the same token and in the same job object are effectively in the same security context. However, one not-well-understood fact is that applications that have windows on the same desktop are also effectively in the same security context because the sending and receiving of window messages is not subject to any security checks. Sending messages across desktops is not allowed. This is the source of the infamous "shatter" attacks, which is why services should not host windows on the interactive desktop. A Windows desktop is a regular kernel object that can be created and assigned a security descriptor.

In a standard Windows installation, at least two desktops are attached to the interactive window station; the regular (default) desktop, and the logon desktop. The sandbox creates a third desktop that is associated to all target processes. This desktop is never visible or interactive and effectively isolates the sandboxed processes from snooping the user's interaction and from sending messages to windows operating at more privileged contexts.

The only disadvantage of an alternate desktop is that it uses approximately 4MB of RAM from a separate pool, possibly more on Vista.

More information about Window Stations

###The integrity levels

Integrity levels are available on Windows Vista and later versions. They don't define a security boundary in the strict sense, but they do provide a form of mandatory access control (MAC) and act as the basis of Microsoft's Internet Explorer sandbox. 

Integrity levels are implemented as a special set of SID and ACL entries representing five levels of increasing privilege: untrusted, low, medium, high, system. Access to an object may be restricted if the object is at a higher integrity level than the requesting token. Integrity levels also implement User Interface Privilege Isolation, which applies the rules of integrity levels to window messages exchanged between different processes on the same desktop.

By default, a token can read an object of a higher integrity level, but not write to it. Most desktop applications run at medium integrity (MI), while less trusted processes like Internet Explorer's protected mode and our own sandbox run at low integrity (LI). A low integrity mode token can access only the following shared resources:
* Read access to most files
* Write access to %USER PROFILE%\AppData\LocalLow
* Read access to most of the registry
* Write access to HKEY_CURRENT_USER\Software\AppDataLow
Clipboard (copy and paste for certain formats)
* Remote procedure call (RPC)
* TCP/IP Sockets
* Window messages exposed via ChangeWindowMessageFilter
* Shared memory exposed via LI (low integrity) labels
* COM interfaces with LI (low integrity) launch activation rights
* Named pipes exposed via LI (low integrity) labels

You'll notice that the previously described attributes of the token, job object, and alternate desktop are more restrictive, and would in fact block access to everything allowed in the above list. So, the integrity level is a bit redundant with the other measures, but it can be seen as an additional degree of defense-in-depth, and its use has no visible impact on performance or resource usage. 

More information on integrity levels can be found at [03].

###Process mitigation policies

Most process mitigation policies can be applied to the target process by means of SetProcessMitigationPolicy.  The sandbox uses this API to set various policies on the target process for enforcing security characteristics.

**Relocate Images:**

* \>= Win8
* Address-load randomization (ASLR) on all images in process (and must be supported by all images).

**Heap Terminate:**

* \>= Win8
* Terminates the process on Windows heap corruption.

**Bottom-up ASLR:**

* \>= Win8
* Sets random lower bound as minimum user address for the process.

**High-entropy ASLR:**

* \>= Win8
* Increases randomness range for bottom-up ASLR to 1TB.

**Strict Handle Checks:**

* \>= Win8
* Immediately raises an exception on a bad handle reference.

**Win32k.sys lockdown:**

* \>= Win8
* ProcessSystemCallDisablePolicy, which allows selective disabling of system calls available from the target process.
* Renderer processes now have this set to DisallowWin32kSystemCalls which means that calls from user mode that are serviced by win32k.sys are no longer permitted. This significantly reduces the kernel attack surface available from a renderer.  See here for more details.

**App Container (low box token):**

* \>= Win8
* In Windows this is implemented at the kernel level by a Low Box token which is a stripped version of a normal token with limited privilege (normally just SeChangeNotifyPrivilege and SeIncreaseWorkingSetPrivilege), running at Low integrity level and an array of "Capabilities" which can be mapped to allow/deny what the process is allowed to do (see [MSDN](https://msdn.microsoft.com/en-us/library/windows/apps/hh464936.aspx) for a high level description). The capability most interesting from a sandbox perspective is denying is access to the network, as it turns out network checks are enforced if the token is a Low Box token and the INTERNET_CLIENT Capability is not present.
* The sandbox therefore takes the existing restricted token and adds the Low Box attributes, without granting any Capabilities, so as to gain the additional protection of no network access from the sandboxed process.

**Disable Font Loading:**

* \>= Win10
* ProcessFontDisablePolicy

**Disable Image Load from Remote Devices:**

* \>= Win10 TH2
* ProcessImageLoadPolicy
* E.g. UNC path to network resource.

**Disable Image Load of "mandatory low" (low integrity level):**

* \>= Win10 TH2
* ProcessImageLoadPolicy
* E.g. temporary internet files.

**Extra Disable Child Process Creation:**

* \>= Win10 TH2
* If the Job level <= JOB_LIMITED_USER, set PROC_THREAD_ATTRIBUTE_CHILD_PROCESS_POLICY to PROCESS_CREATION_CHILD_PROCESS_RESTRICTED via UpdateProcThreadAttribute().
* This is an extra layer of defense, given that Job levels can be broken out of. [REF: [ticket](https://bugs.chromium.org/p/project-zero/issues/detail?id=213&redir=1), [Project Zero blog](http://googleprojectzero.blogspot.co.uk/2015/05/in-console-able.html).]

###Other caveats

The operating system might have bugs. Of interest are bugs in the Windows API that allow the bypass of the regular security checks. If such a bug exists, malware will be able to bypass the sandbox restrictions and broker policy and possibly compromise the computer. Under Windows, there is no practical way to prevent code in the sandbox from calling a system service.

In addition, third party software, particularly anti-malware solutions, can create new attack vectors. The most troublesome are applications that inject dlls in order to enable some (usually unwanted) capability. These dlls will also get injected in the sandbox process. In the best case they will malfunction, and in the worst case can create backdoors to other processes or to the file system itself, enabling specially crafted malware to escape the sandbox. 

##Sandbox policy

The actual restrictions applied to a target process are configured by a policy. The policy is just a programmatic interface that the broker calls to define the restrictions and allowances. Four functions control the restrictions, roughly corresponding to the four Windows mechanisms:
* TargetPolicy::SetTokenLevel()
* TargetPolicy::SetJobLevel()
* TargetPolicy::SetIntegrityLevel()
* TargetPolicy::SetDesktop()

The first three calls take an integer level parameter that goes from very strict to very loose; for example, the token level has 7 levels and the job level has 5 levels. Chromium renderers are typically run with the most strict level in all four mechanisms. Finally, the last (desktop) policy is binary and can only be used to indicate if a target is run on an alternate desktop or not.

The restrictions are by design coarse in that they affect all securable resources that the target can touch, but sometimes a more finely-grained resolution is needed. The policy interface allows the broker to specify exceptions. An exception is a way to take a specific Windows API call issued in the target and proxy it over to the broker. The broker can inspect the parameters and re-issue the call as is, re-issue the call with different parameters, or simply deny the call. To specify exceptions there is a single call: AddRule. The following kinds of rules for different Windows subsystems are supported at this time:
* Files
* Named pipes
* Process creation
* Registry
* Synchronization objects

The exact form of the rules for each subsystem varies, but in general rules are triggered based on a string pattern. For example, a possible file rule is:
```c
AddRule(SUBSYS_FILES, FILES_ALLOW_READONLY, L"c:\\temp\\app_log\\d*.dmp")
```
This rule specifies that access will be granted if a target wants to open a file, for read-only access as long as the file matches the pattern expression; for example c:\temp\app_log\domino.dmp is a file that satisfies the pattern. Consult the header files for an up-to-date list of supported objects and supported actions.

Rules can only be added before each target process is spawned, and cannot be modified while a target is running, but different targets can have different rules.

##Target bootstrapping

Targets do not start executing with the restrictions specified by policy. They start executing with a token that is very close to the token the regular user processes have. The reason is that during process bootstrapping the OS loader accesses a lot of resources, most of them are actually undocumented and can change at any time. Also, most applications use the standard CRT provided with the standard development tools; after the process is bootstrapped the CRT needs to initialize as well and there again the internals of the CRT initialization are undocumented.

Therefore, during the bootstrapping phase the process actually uses two tokens: the lockdown token which is the process token as is and the initial token which is set as the impersonation token of the initial thread. In fact the actual SetTokenLevel definition is:
```c
SetTokenLevel(TokenLevel initial, TokenLevel lockdown)
```
After all the critical initialization is done, execution continues at main() or WinMain(), here the two tokens are still active, but only the initial thread can use the more powerful initial token. It is the target's responsibility to discard the initial token when ready. This is done with a single call:
```
LowerToken()
```
After this call is issued by the target the only token available is the lockdown token and the full sandbox restrictions go into effect. The effects of this call cannot be undone. Note that the initial token is a impersonation token only valid for the main thread, other threads created in the target process use only the lockdown token and therefore should not attempt to obtain any system resources subject to a security check.

The fact that the target starts with a privileged token simplifies the explicit policy since anything privileged that needs to be done once, at process startup can be done before the LowerToken() call and does not require to have rules in the policy.

>**Important**

> Make sure any sensitive OS handles obtained with the initial token are closed before calling LowerToken(). Any leaked handle can be abused by malware to escape the sandbox.


##References

[01] Richter, Jeffrey "Make Your Windows 2000 Processes Play Nice Together With Job Kernel Objects" 

http://www.microsoft.com/msj/0399/jobkernelobj/jobkernelobj.aspx

[02] Brown, Keith "What Is a Token" (wiki) 

http://alt.pluralsight.com/wiki/default.aspx/Keith.GuideBook/WhatIsAToken.htm

[03] Windows Integrity Mechanism Design (MSDN)

http://msdn.microsoft.com/en-us/library/bb625963.aspx