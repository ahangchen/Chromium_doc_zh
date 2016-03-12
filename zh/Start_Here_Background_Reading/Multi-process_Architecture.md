#多进程架构
这个文档描述了Chromium的高层架构

##问题

构建一个从不会挂起或崩溃的渲染引擎几乎是不可能的。构建一个完全安全的渲染引擎也是几乎不可能的。

在某种程度上，web浏览器当前状态就像一个与过去的多任务操作系统合作的单独的用户。正如在一个这样的操作系统中的错误程序会让整个系统挂掉，所以一个错误的web页面也可以让一个现代浏览器挂掉。仅仅需要一个浏览器或插件的bug，就饿能让整个浏览器和所有正在运行的标签页停止运行。

现代操作系统更加鲁棒，因为他们把应用程序分成了彼此隔离的独立线程。一个程序中的crash通常不会影响其他程序或整个操作系统，每个用户对用户数据的访问也是有限制的。

##架构概览

我们为浏览器的标签页使用独立的进程，以此保护整个应用程序免受渲染引擎中的bug和故障的伤害。我们也会限制每个渲染引擎进程的相互访问，以及他们与系统其他部分的访问。某些程度上，这为web浏览提供了内存保护，为操作系统提供了访问控制。

我们把运行UI的进程叫做主进程（main），把插件进程称为“浏览器进程”或“浏览器（Browser）”。相似的，标签页相关的进程被称作“渲染线程”或“渲染器（renderer）”。渲染器使用[WebKit](http://webkit.org/)开源引擎来实现中断与html的布局。

![img](../../arch.png)

###管理渲染进程

每个渲染进程有一个全局的RenderProcess对象，管理它与父浏览器进程之间的通信，维护全局的状态。浏览器为每个渲染进程维护一个对应的RenderViewHost，用来管理浏览器状态，并与渲染器交流。浏览器与渲染器使用[Chromium's IPC system](../General_Architecture/Inter-process_Communication.md)进行交流。

###管理view

每个渲染进程有一个以上的RenderView对象，由RenderProcess管理（它与标签页的内容相关）。对应的RenderProcessHost维护一个与渲染器中每个view相关的RenderViewHost。每个view被赋予一个view ID,以区分同一个渲染器中的不同view。这些ID在每个渲染器内是唯一的，但在浏览器中不是，所以区分一个view需要一个RenderProcessHost和一个view ID。

浏览器与一个包含内容的特定标签页之间的交流是通过这些RenderViewHost对象来完成的，它们知道如何通过他们的RenderProcessHost向RenderProcess和RenderView送消息。

##组件与接口

在渲染进程中：

- *RenderProcess*处理与浏览器中对应的*RenderProcessHost*的通信。每个渲染进程就有唯一的一个RenderProcess对象。这就是所有浏览器-渲染器之间的交互发生的方式。

- RenderView对象与它在浏览器进程中对应的RenderViewHost和我们的webkit嵌入层通信（通过RenderProcess）。这个对象代表了一个网页在标签页或一个弹出窗口的内容。

在浏览器进程中:

- Browser对象代表了顶级浏览器窗口
- The RenderProcessHost object represents the browser side of a single browser ↔ renderer IPC connection. There is one RenderProcessHost in the browser process for each render process.
- The RenderViewHost object encapsulates communication with the remote RenderView, and RenderWidgetHost handles the input and painting for RenderWidget in the browser.

For more detailed information on how this embedding works, see the [How Chromium displays web pages design document](How_Chromium_displays_web_pages_design_document).

##Sharing the render process

In general, each new window or tab opens in a new process. The browser will spawn a new process and instruct it to create a single *RenderView*.

Sometimes it is necessary or desirable to share the render process between tabs or windows. A web application opens a new window that it expects to communicate with synchronously, for example, using window.open in JavaScript. In this case, when we create a new window or tab, we need to reuse the process that the window was opened with. We also have strategies to assign new tabs to existing processes if the total number of processes is too large, or if the user already has a process open navigated to that domain. These strategies are described in [Process Models](../General_Architecture/Process_Models.md).

##Detecting crashed or misbehaving renderers

Each IPC connection to a browser process watches the process handles. If these handles are signaled, the render process has crashed and the tabs are notified of the crash. For now, we show a "sad tab" screen that notifies the user that the renderer has crashed. The page can be reloaded by pressing the reload button or by starting a new navigation. When this happens, we notice that there is no process and create a new one.

##Sandboxing the renderer

Given Webkit is running in a separate process, we have the opportunity to restrict its access to system resources. For example, we can ensure that the renderer's only access to the network is via its parent browser process. Likewise, we can restrict its access to the filesystem using the host operating system's built-in permissions.

In addition to restricting the renderer's access to the filesystem and network, we can also place limitations on its access to the user's display and related objects. We run each render process on a separate Windows "Desktop" which is not visible to the user. This prevents a compromised renderer from opening new windows or capturing keystrokes.

##Giving back memory

Given renderers running in separate processes, it becomes straightforward to treat hidden tabs as lower priority. Normally, minimized processes on Windows have their memory automatically put into a pool of "available memory." In low-memory situations, Windows will swap this memory to disk before it swaps out higher-priority memory, helping to keep the user-visible programs more responsive. We can apply this same principle to hidden tabs. When a render process has no top-level tabs, we can release that process's "working set" size as a hint to the system to swap that memory out to disk first if necessary. Because we found that reducing the working set size also reduces tab switching performance when the user is switching between two tabs, we release this memory gradually. This means that if the user switches back to a recently used tab, that tab's memory is more likely to be paged in than less recently used tabs. Users with enough memory to run all their programs will not notice this process at all: Windows will only actually reclaim such data if it needs it, so there is no performance hit when there is ample memory.

This helps us get a more optimal memory footprint in low-memory situations. The memory associated with seldom-used background tabs can get entirely swapped out while foreground tabs' data can be entirely loaded into memory. In contrast, a single-process browser will have all tabs' data randomly distributed in its memory, and it is impossible to separate the used and unused data so cleanly, wasting both memory and performance.

##Plug-ins

Firefox-style NPAPI plug-ins run in their own process, separate from renderers. This is described in detail in [Plugin Architecture](../General_Architecture/Plugin_Architecture.md).

##How to Add New Features (without bloating RenderView/RenderViewHost/WebContents)
###Problem

Historically, new features (i.e. autofill to pick an example) have been added by bolting on their code to RenderView in the renderer process, and RenderViewHost in the browser process. If a feature was handled on the IO thread in the browser process, then its IPC messages were dispatched in BrowserMessageFilter. RenderViewHost would often dispatch the IPC message only to call WebContents (through the RenderViewHostDelegate interface), which would then call to another piece of code. All the IPC messages between the browser and renderer were declared in a massive render_messages_internal.h.  Touching each of these files for every feature meant that the classes became bloated.

###Solution

We have added helper classes and mechanisms for filtering IPC messages on each of the above threads. This makes it easier to write self contained features.

####Renderer side:

If you want to filter and send IPC messages, implement the RenderViewObserver interface (content/renderer/render_view_observer.h). The RenderViewObserver base class takes a RenderView and manages the object's lifetime so that it's tied to RenderView (this is overridable). The class can then filter and send IPC messages, and additionally gets notification about frame loading and closing, which many features need.  As an example, see ChromeExtensionHelper (chrome/renderer/extensions/chrome_extension_helper.h).

If your feature has part of the code in WebKit, avoid having callbacks go through WebViewClient interface so that we don't bloat it. Consider creating a new WebKit interface that the WebKit code calls, and have the renderer side class implement it. As an example, see WebAutoFillClient (WebKit/chromium/public/WebAutoFillClient.h).

###Browser UI thread:

The WebContentsObserver (content/public/browser/web_contents_observer.h) interface allows objects on the UI thread to filter IPC messages and also gives notifications about frame navigation. As an example, see TabHelper (chrome/browser/extensions/tab_helper.h).

###Browser other threads:

To filter and send IPC messages on other browser threads, such as IO/FILE/WEBKIT etc, implement BrowserMessageFilter interface (content/browser/browser_message_filter.h). The BrowserRenderProcessHost object creates and adds the filters in its CreateMessageFilters function.

In general, if a feature has more than a few IPC messages, they should be moved into a separate file (i.e. not be added to render_messages_internal.h). This also helps with filtering messages on a thread other than the IO thread. As an example, see content/common/pepper_file_messages.h. This allows their filter, PepperFileMessageFilter, to easily send the messages to the file thread without having to specify their IDs in multiple places.
```c
void PepperFileMessageFilter::OverrideThreadForMessage(
    const IPC::Message& message,
    BrowserThread::ID* thread) {
  if (IPC_MESSAGE_CLASS(message) == PepperFileMsgStart)
    *thread = BrowserThread::FILE;
}
```
