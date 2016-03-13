#Chromium如何展示网页

这个文档从底层描述了Chromium是如何展示网页的。请确认你已经读过[多进程架构](Multi-process_Architecture.md)这篇文章。你会特别想要了解主要组件的框架。你也可能对[多进程资源加载](General_Architecture/Multi-process_Resource_Loading.md)感兴趣，以了解网页是如何从网络中获取到的。

##应用概念层

![img](../layer.png)

(关于这个阐述的原始Google文档是http://goo.gl/MsEJX，开放给所有@chromium.org的人编辑)

每个矩形代表了一个应用概念层，每一层都不了解上一层，也对上一层没有依赖。


- WebKit：Safari，Chromium和其他所有基于WebKit的浏览器共享的渲染引擎。WebKit Port是WebKit的一个部分，用来集成平台独立的系统服务，比如资源加载与图像。

- Glue：将WebKit的类型转为Chromium的类型。这就是我们的“WebKit嵌入层”。这是两个browser，Chromium，和test_shell（允许我们测试WebKit）的基础。

- Renderer / Render host： 这是Chromium的“多进程嵌入层”。它代理通知，并跨过进程边界执行指令。

- WebContents：一个可重用的组件，是内容模块的主类。它易于嵌入，允许多进程将HTML绘制成View。查看[content module pages](Other/content_module___content_API.md)以获得更多信息。

- Browser: 代表浏览器窗口，包含多个WebContent。

- Tab Helpers：可以被绑定到WebContent的独立对象（通过WebContentsUserData混杂）。浏览器将这些独立对象中的一种绑定到WebContent给它持有，一个给网站图标，一个给信息栏，等等。

##WebKit

我们使用WebKit开源工程来布局web页面。这部分代码是从Apple中pull过来的，存储在/third_party/WebKit目录。WebKit主要由“WebCore”组成，这代表了核心的布局功能，还有“JavaScriptCore”，这被用来运行JavaScript。我们只在测试时运行JavaScriptCore，通常情况下，我们用我们自己高性能的V8 Javascript引擎来代替它。事实上，我们不完全是使用Apple称之为“WebKit”的那一层，这是WebCore和OS X应用程序（比如Safari）之间的嵌入API。为了方便，我们通常把从Apple学到的代码称为“WebKit”。

###The WebKit port

在最低层，我们有我们的WebKit “port”。这是我们对于需要的平台相关功能的实现，它们与平台无关的WebCore代码交互。这些文件在WebKit树上，通常在chromium目录，或以Chromium为后缀的文件中。我们的port中的大部分其实是与操作系统无关的：你可以把它认为WebCore的“Chromium port”。但某些方面，比如字体渲染，必须在不同平台上做不同的处理。

- 网络交流由我们的[多进程资源加载](General_Architecture/Multi-process_Resource_Loading.md)系统处理，而非直接从渲染线程跳到操作系统处理
- 图像使用了为Android开发的Skia图形库。这是一个跨平台的图形库，处理所有的图形和图像，除了文本。Skia在/third_party/skia里。图形操作的主要入口是/webkit/port/platform/graphics/GraphicsContextSkia.cpp。它在这个目录里，使用了许多其他的文件，还有那些/base/gfx里的文件。

###The WebKit glue（胶水）

The Chromium application uses different types, coding styles, and code layout than the third-party WebKit code. The WebKit "glue" provides a more convenient embedding API for WebKit using Google coding conventions and types (for example, we use std::string instead of WebCore::String and GURL instead of KURL). The glue code is located in /webkit/glue. The glue objects are typically named similar to the WebKit objects, but with "Web" at the beginning. For example, WebCore::Frame becomes WebFrame.
The WebKit "glue" layer insulates the rest of the Chromium code base from WebCore data types to help minimize the impact of WebCore changes on the Chromium code base. As such, WebCore data types are never used directly by Chromium. APIs are added to the WebKit "glue" for the benefit of Chromium when it needs to poke at some WebCore object.

The "test shell" application is a bare-bones web browser for testing our WebKit port and glue code. It uses the same glue interface for communicating with WebKit as Chromium does. It provides a simpler way for developers to test new code without having many complicated browser features, threads, and processes. This application is also used to run the automated WebKit tests. However, the downside of the "test shell" is that it doesn't exercise WebKit as Chromium does, in a multi-process way. The content module is embedded in an application called "content shell" which will soon be running the tests instead.

##The render process
![img](../Renderingintherenderer-v2.png)

Chromium's render process embeds our WebKit port using the glue interface. It does not contain very much code: its job is primarily to be the renderer side of the [IPC](General_Architecture) channel to the browser..
The most important class in the renderer is the RenderView, located in /content/renderer/render_view_impl.cc. This object represents a web page. It handles all navigation-related commands to and from the browser process. It derives from RenderWidget which provides painting and input event handling. The RenderView communicates with the browser process via the global (per render process) RenderProcess object.

**FAQ: What's the difference between RenderWidget and RenderView?** RenderWidget maps to one WebCore::Widget object by implementing the abstract interface in the glue layer called WebWidgetDelegate.. This is basically a Window on the screen that receives input events and that we paint into. A RenderView inherits from RenderWidget and is the contents of a tab or popup Window. It handles navigational commands in addition to the painting and input events of the widget. There is only one case where a RenderWidget exists without a RenderView, and that's for select boxes on the web page. These are the boxes with the down arrows that pop up a list of options. The select boxes must be rendered using a native window so that they can appear above everything else, and pop out of the frame if necessary. These windows need to paint and receive input, but there isn't a separate "web page" (RenderView) for them.

###Threads in the renderer

Each renderer has two threads (see the [multi-process architecture](Multi-process_Architecture.md) page for a diagram, or threading in Chromium for how to program with them). The render thread is where the main objects such as the RenderView and all WebKit code run. When it communicates to the browser, messages are first sent to the main thread, which in turn dispatches the message to the browser process. Among other things, this allows us to send messages synchronously from the renderer to the browser. This happens for a small set of operations where a result from the browser is required to continue. An example is getting the cookies for a page when requested by JavaScript. The renderer thread will block, and the main thread will queue all messages that are received until the correct response is found. Any messages received in the meantime are subsequently posted to the renderer thread for normal processing.
##The browser process

![img](../rendering_browser.png)

###Low-level browser process objects

All [IPC](General_Architecture) communication with the render processes is done on the I/O thread of the browser. This thread also handles all [network communication](Multi-process_Resource_Loading.md) which keeps it from interfering with the user interface.

When a RenderProcessHost is initialized on the main thread (where the user interface runs), it creates the new renderer process and a ChannelProxy [IPC](General_Architecture) object with a named pipe to the renderer. This object runs on the I/O thread of the browser, listening to the named pipe to the renderer, and automatically forwards all messages back to the RenderProcessHost on the UI thread. A ResourceMessageFilter will be installed in this channel which will filter out certain messages that can be handled directly on the I/O thread such as network requests. This filtering happens in ResourceMessageFilter::OnMessageReceived.

The RenderProcessHost on the UI thread is responsible for dispatching all view-specific messages to the appropriate RenderViewHost (it handles a limited number of non-view-specific messages itself). This dispatching happens in RenderProcessHost::OnMessageReceived.

###High-level browser process objects

View-specific messages come into RenderViewHost::OnMessageReceived. Most of the messages are handled here, and the rest get forwarded to the RenderWidgetHost base class. These two objects map to the RenderView and the RenderWidget in the renderer (see "The Render Process" above for what these mean). Each platform has a view class (RenderWidgetHostView[Aura|Gtk|Mac|Win]) to implement integration into the native view system.

Above the RenderView/Widget is the WebContents object, and most of the messages actually end up as function calls on that object. A WebContents represents the contents of a webpage. It is the top-level object in the content module, and has the responsibility of displaying a web page in a rectangular view. See the [content module pages](Other/content_module___content_API.md) for more information.

The WebContents object is contained in a TabContentsWrapper. That is in chrome/ and is responsible for a tab.

##Illustrative examples

Additional examples covering navigation and startup are in [Getting Around the Chromium Source Code](https://www.chromium.org/developers/how-tos/getting-around-the-chrome-source-code).
###Life of a "set cursor" message

Setting the cursor is an example of a typical message that is sent from the renderer to the browser. In the renderer, here is what happens.

- Set cursor messages are generated by WebKit internally, typically in response to an input event. The set cursor message will start out in RenderWidget::SetCursor in content/renderer/render_widget.cc.

- It will call RenderWidget::Send to dispatch the message. This method is also used by RenderView to send messages to the browser. It will call RenderThread::Send.

- This will call the IPC::SyncChannel which will internally proxy the message to the main thread of the renderer and post it to the named pipe for sending to the browser.

Then the browser takes control:

- The IPC::ChannelProxy in the RenderProcessHost receives all message on the I/O thread of the browser. It first sends them through the ResourceMessageFilter that dispatches network requests and related messages directly on the I/O thread. Since our message is not filtered out, it continues on to the UI thread of the browser (the IPC::ChannelProxy does this internally).

- RenderProcessHost::OnMessageReceived in content/browser/renderer_host/render_process_host_impl.cc gets the messages for all views in the corresponding render process. It handles several types of messages directly, and for the rest forwards to the appropriate RenderViewHost corresponding to the source RenderView that sent the message.

- The message arrives at RenderViewHost::OnMessageReceived in content/browser/renderer_host/render_view_host_impl.cc. Many messages are handled here, but ours is not because it's a message sent from the RenderWidget and handled by the RenderWidgetHost.

- All unhandled messages in RenderViewHost are automatically forwarded to the RenderWidgetHost, including our set cursor message.

- The message map in content/browser/renderer_host/render_view_host_impl.cc finally receives the message in RenderWidgetHost::OnMsgSetCursor and calls the appropriate UI function to set the mouse cursor.

###Life of a "mouse click" message

Sending a mouse click is a typical example of a message going from the browser to the renderer.

- The Windows message is received on the UI thread of the browser by RenderWidgetHostViewWin::OnMouseEvent which then calls ForwardMouseEventToRenderer in the same class.

- The forwarder function packages the input event into a cross-platform WebMouseEvent and ends up sending it to the RenderWidgetHost it is associated with.

- RenderWidgetHost::ForwardInputEvent creates an IPC message ViewMsg_HandleInputEvent, serializes the WebInputEvent to it, and calls RenderWidgetHost::Send.

- This just forwards to the owning RenderProcessHost::Send function, which in turn gives the message to the IPC::ChannelProxy.

- Internally, the IPC::ChannelProxy will proxy the message to the I/O thread of the browser and write it to the named pipe of the corresponding renderer.

Note that many other types of messages are created in the WebContents, especially navigational ones. These follow a similar path from the WebContents to the RenderViewHost.

Then the renderer takes control:

- IPC::Channel on the main thread of the renderer reads the message sent by the browser, and IPC::ChannelProxy proxies to the renderer thread.

- RenderView::OnMessageReceived gets the message. Many types messages are handled here directly. Since the click message is not, it falls through (with all other unhandled messages) to RenderWidget::OnMessageReceived which in turn forwards it to RenderWidget::OnHandleInputEvent.

- The input event is given to WebWidgetImpl::HandleInputEvent where it is converted to a WebKit PlatformMouseEvent class and given to the WebCore::Widget class inside WebKit.