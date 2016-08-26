#多进程架构
这个文档描述了Chromium的高层架构

##问题

构建一个从不会挂起或崩溃的渲染引擎几乎是不可能的。构建一个完全安全的渲染引擎也是几乎不可能的。

在某种程度上，web浏览器当前状态就像一个与过去的多任务操作系统合作的单独的用户。正如在一个这样的操作系统中的错误程序会让整个系统挂掉，所以一个错误的web页面也可以让一个现代浏览器挂掉。仅仅需要一个浏览器或插件的bug，就饿能让整个浏览器和所有正在运行的标签页停止运行。

现代操作系统更加鲁棒，因为他们把应用程序分成了彼此隔离的独立线程。一个程序中的crash通常不会影响其他程序或整个操作系统，每个用户对用户数据的访问也是有限制的。

##架构概览

我们为浏览器的标签页使用独立的进程，以此保护整个应用程序免受渲染引擎中的bug和故障的伤害。我们也会限制每个渲染引擎进程的相互访问，以及他们与系统其他部分的访问。某些程度上，这为web浏览提供了内存保护，为操作系统提供了访问控制。

我们把运行UI的进程叫做主进程（main），把插件进程称为“浏览器进程”或“浏览器（Browser）”。相似的，标签页相关的进程被称作“渲染线程”或“渲染器（renderer）”。渲染器使用[WebKit](http://webkit.org/)开源引擎来实现中断与html的布局。

![img](../arch.png)

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
- RenderProcessHost对象代表了浏览器端浏览器的与渲染器的IPC连接。在浏览器进程里，每个渲染进程有一个RenderProcessHost对象。
- RenderViewHost对象封装了与远端浏览器的交流，RenderWidgetHost处理输入并在浏览器中为RenderWidget进行绘制。

想要得到更多关于这种嵌入是如何工作的详细信息，可以查看[How Chromium displays web pages design document](How_Chromium_displays_web_pages_design_document)。


##共享绘制器进程

通常，每个新的window或标签页是在一个新进程里打开的。浏览器会生成一个新的进程，然后指导它去创建一个*RenderView*。

有时候，有这样一种必要或欲望在标签页或窗口间共享渲染进程。一个web应用程序会在期望同步交流时，打开一个新的窗口，比如，在javascript里使用window.open。这种情况下，当我们创建一个新的window或标签页时，我们需要重用打开这个window的进程。我们也有一些策略来把新的标签页分配的已有的进程（如果总的进程数太大的话，或者如果用户已经为这个域名打开了一个进程）。这些策略在[Process Models](../General_Architecture/Process_Models.md)里也有阐述。


##检测crash或者失误的渲染

每个到浏览器进程的IPC连接会观察进程句柄。如果这些句柄是signaled（有信号的），那么渲染进程已经挂了，标签页会得到一个通知。从这时开始，我们会展示一个“sad tab”画面来通知用户渲染器已经挂掉了。这个页面可以按刷新按钮或者通过打开一个新的导航来重新加载。这时，我们会注意到没有对应的进程，然后创建一个新的。

##渲染器中的沙箱

给定的WebKit是运行在独立的进程中的，所以我们有机会限制它对系统资源的访问。例如，我们可以确保渲染器唯一的网络权限是通过它的父浏览器进程实现。相似的，我们可以限制它对文件系统的访问权限来使用host操作系统内置的权限。

除了限制渲染器对文件系统和网络的访问权限，我们也可以限制它对用户的显示器以及相关的东西的一些权限。我们在独立的windows桌面（对用户不可见）中运行每个进程。这避免了让渲染器在新的标签页或捕捉按键之间妥协。

##归还内存

让渲染器运行在独立的进程中，赋予隐藏的标签页更低的优先级会更加直接。通常，Windows平台上的最小化的进程会把它们的内存自动放到一个“可用内存”池里。在低内存的情况下，Windows会在交换这部分内存到更高优先级内存前，把它们交换到磁盘，以保证用户可见的程序更易响应。我们可以对隐藏的标签页使用相同的策略。当渲染器进程没有顶层标签页时，我们可以释放进程的“工作集”空间，作为一个给系统的信号，让它如果必要的话，优先把这些内存交换到磁盘。因为我们发现，当用户在两个标签页间切换时，减少工作集大小也会减少标签页切换性能，所以我们是逐渐释放这部分内存的。这意味着如果用户切换回最近使用的标签页，这个标签页的内存比最近较少访问的标签页更可能被换入。有着足够内存的用户运行他们所有的程序时根本不会注意到这个进程：事实上Windows只会在需要的时候重新声明这块数据，所以在有充分内存时，不会有性能瓶颈。

这能帮助我们在低内存的情况下得到最佳的内存轨迹。几乎不被使用的后台标签页相关的内存可以被完全交换掉，前台标签页的数据可以被完全加载进内存。相反的，一个单进程浏览器会在它的内存里随机分配所有标签页的数据，并且不可能如此清晰地隔离已使用的和未使用的数据，导致了内存和性能上的浪费。

##插件

Firefox风格的NPAPI插件运行在他们自己的进程里，与渲染器隔离。这会在[Plugin Architecture](../General_Architecture/Plugin_Architecture.md)中描述。

##如何添加新特性(不用扩充RenderView/RenderViewHost/WebContents)
###问题

过去，新的特性（比如，自动填充选取样例）可以通过把新特性的代码导入到RenderView类（在渲染器进程里）和RenderViewHost类（在浏览器进程里）。如果一个新的特性是在浏览器进程的IO线程里处理的，那么它的IPC信息由BrowserMessageFilter调度。RenderViewHost会只为了调用WebContent对象进程调用IPC信息，这会调用另一块代码。所有的浏览器与渲染器之间的IPC信息会被声明在一个巨大的render_messages_internal.h里，为每个新特性修改所有的这些文件意味着这些类会变得臃肿。


###解决方案

我们增加了helper类和对上面的每个线程IPC信息的过滤的机制。这使得编写自洽的特性更加容易。

####渲染器端

如果你想要过滤和发送IPC信息，实现RenderViewObserver接口(content/renderer/render_view_observer.h)。RenderViewObserver基类持有一个RenderView类，管理对象的生命周期，使其绑定到RenderView（它是可重写的）。这个类就可以过滤和发送IPC消息，此外还可以获得许多特性需要的关于页面加载与关闭的通知。作为一个例子，可以查看ChromeExtensionHelper (chrome/renderer/extensions/chrome_extension_helper.h)。

如果你的特性有一部分代码是在WebKit内的，避免通过WebViewClient接口回调，这样我们就不会使得WebViewClient变得庞大。考虑创建新的WebKit接口给WebKit代码调用，让渲染器端的类去实现它。作为一个例子，查看WebAutoFillClient (WebKit/chromium/public/WebAutoFillClient.h).

###浏览器UI线程

WebContentsObserver (content/public/browser/web_contents_observer.h)接口允许UI线程的对象过滤IPC信息，以及给出关于页面导航的通知。作为一个例子：查看TabHelper (chrome/browser/extensions/tab_helper.h)。

###浏览器其他线程

为了过滤和发送IPC信息给其他的浏览器线程，比如IO/FILE/WEBKIT等等，实现BrowserMessageFilter接口(content/browser/browser_message_filter.h)。BrowserRenderProcessHost对象在它的CreateMessageFilters函数里创造和增加过滤器。

通常，如果一个特性有许多IPC消息，这些消息应该移动到一个独立的文件（例如，不要加到render_messages_internal.h里）。这对过滤线程（除了IO线程）也有帮助。作为一个例子，查看content/common/pepper_file_messages.h。这允许他们的过滤器PepperFileMessageFilter方便的发送文件到file线程，而不用在很多位置指定它们的ID。
```c
void PepperFileMessageFilter::OverrideThreadForMessage(
    const IPC::Message& message,
    BrowserThread::ID* thread) {
  if (IPC_MESSAGE_CLASS(message) == PepperFileMsgStart)
    *thread = BrowserThread::FILE;
}
```
