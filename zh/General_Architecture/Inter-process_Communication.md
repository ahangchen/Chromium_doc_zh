#跨进程通信 (IPC)
##概览

Chromium有一个[多进程架构](Start_Here_Background_Reading/Multi-process_Architecture.md)，这意味着我们有许多需要互相交流的进程。我们的主要跨进程交流元素是命名管道。在Linux和OS X上，我们使用socketpair()。每个渲染器进程可以分配到一个命名管道来跟浏览器进程交流。这些管道是用异步方式使用的，确保没有哪个端会等待另一个端。

想要得到如何编写安全的IPC端点的知识，请查看[IPC安全要点](https://www.chromium.org/Home/chromium-security/education/security-tips-for-ipc).

###浏览器中IPC

在浏览器中，与渲染器的交流是通过一个独立的I/O线程完成的。来自或者去往view的消息需要使用一个ChannelProxy代理到主线程。这种方案的优点是，资源请求（比如网页等），这种最经常且极其关注性能的消息，可以整个的在I/O线程中处理，不会阻塞用户界面。这些通过使用Channel::MessageFilter（由RenderProcessHost插入channel）来完成。这个过滤器运行在I/O线程里，拦截资源请求信息，将它们直接转发到资源分发主机。查看[多进程资源加载](General_Architecture/Multi-process/Multi-process_Resource_Loading.md)获取更多关于资源加载的信息。

###渲染器中的IPC

每个渲染器也有一个线程管理交流（在这个例子里，是主线程），而大多数渲染和大多数处理发生在另一个线程里（查看多进程架构的那个图表）。大多数消息通过主渲染线程从浏览器发送给WebKit线程，反之亦然。这个额外的线程是用于支持同步的渲染器到浏览器的消息（参考下面的“同步消息”）。



##消息

###消息的类型

我们有两种基本类型的消息：”路由“和”控制“。控制消息由创建管道的类处理，有时候这个类允许其他人通过一个MessageRouter对象接收消息，其他监听器可以通过这个对象注册和接收有着唯一管道id的消息。

例如，渲染时，控制消息没有消息指定目标view，会被RenderProcess（渲染器）或者RenderProcessHost（浏览器）处理。来自资源的请求或者修改剪贴板的请求是没有目标view的，所以是控制消息。一个路由消息的例子是要求一个view画一个区域消息。

路由消息曾经被用于从指定的RenderViewHost获取消息。然而，技术上，任何类可以通过使用RenderProcessHost::GetNextRoutingID接收路由消息，并用RenderProcessHost::AddRoute注册它自己这个消息。现在，RenderFrameHost和RenderViewHost有了他们自己的路由ID了。

消息是否是独立类型在于，消息是从浏览器发送到渲染器，还是从渲染器到浏览器。从浏览器到渲染器的被称为View消息，因为它们被发送给RenderViewHost。从渲染器发送到浏览器的消息叫做ViewHost消息，因为他们被发送给RenderViewHost。你会注意到render_messages_internal.h里定义的消息被分为两类。

插件也有独立的进程。像渲染消息那样，PluginProcess消息（从浏览器发送到插件进程）和PluginProcessHost消息（从插件进程发送到浏览器）。这些消息都定义在plugin_messages_internal.h里。自动化消息（用于控制浏览器做UI测试）通过相同的方式完成。

###声明消息

特殊的宏用于声明消息。渲染器和浏览器间发送的消息都声明在render_messages_internal.h里。有两个部分，一个给发送到渲染器的View消息，一个给发送到浏览器的ViewHost消息。

如果要声明一个从渲染器发送到浏览器（一个ViewHost消息），并且指定一个view（路由）包含一个url和一个整数作为参数，这样写：

```c++
IPC_MESSAGE_ROUTED2(ViewHostMsg_MyMessage, GURL, int)
```

如果要声明一个从浏览器发往渲染器的控制消息（一个View消息），并且不指定目标view（控制），不包含参数，这样写：

```c++
IPC_MESSAGE_CONTROL0(ViewMsg_MyMessage)
```
####Pickling values

参数通过ParamTraits模板序列化或者反序列化到消息体中。这种模板的具体化在ipc_message_utils.h中提供给大多数常见的类型。如果你定义了你自己的类型，你也需要为它定义你自己的ParamTraits具体形式。

有时候，一条消息有太多的值了，没法合理得放到消息里。这种情况下，我们定义一个独立的结构来存放这些值。例如，对于FrameMsg_Navigate消息，在[navigation_params.h](https://code.google.com/p/chromium/codesearch/#chromium/src/content/common/navigation_params.h)中，定义了CommonNavigationParams结构。[frame_messages.h](https://code.google.com/p/chromium/codesearch/#chromium/src/content/common/frame_messages.h)用[IPC_STRUCT_TRAITS](https://code.google.com/p/chromium/codesearch/#chromium/src/ipc/param_traits_macros.h)类型的宏定义了这个结构的具体ParamTraits。


###发送消息

你通过“channel（通道）”发送消息（往下看）。在浏览器里，RenderProcessHost包含了用于从浏览器UI线程发送消息到渲染器的channel。为了方便，RenderWidgetHost（RenderViewHost的基类）提供了一个Send函数。

消息由指针发送，并将在它们分发后由IPC层删除。因此，一旦你发现合适的Send函数，尽管带着一条新消息去调用它：
```c++
Send(new ViewMsg_StopFinding(routing_id_));
```

注意，你必须按顺序指定路由ID，让消息能够路由到接收端正确的View/ViewHost。RenderWidgetHost(RenderViewHost的基类)和RenderWidget（RenderViewHost的基类）都有GetRoutingID()成员函数给你使用。


###处理消息

Messages are handled by implementing the IPC::Listener interface, the most important function on which is OnMessageReceived. We have a variety of macros to simplify message handling in this function, which can best be illustrated by example:

```c++
MyClass::OnMessageReceived(const IPC::Message& message) {
  IPC_BEGIN_MESSAGE_MAP(MyClass, message)
    // Will call OnMyMessage with the message. The parameters of the message will be unpacked for you.
    IPC_MESSAGE_HANDLER(ViewHostMsg_MyMessage, OnMyMessage)  
    ...
    IPC_MESSAGE_UNHANDLED_ERROR()  // This will throw an exception for unhandled messages.
  IPC_END_MESSAGE_MAP()
}

// This function will be called with the parameters extracted from the ViewHostMsg_MyMessage message.
MyClass::OnMyMessage(const GURL& url, int something) {
  ...
}
```

You can also use IPC_DEFINE_MESSAGE_MAP to implement the function definition for you as well. In this case, do not specify a message variable name, it will declare a OnMessageReceived function on the given class and implement its guts.

Other macros:

- IPC_MESSAGE_FORWARD: This is the same as IPC_MESSAGE_HANDLER but you can specify your own class to send the message to, instead of sending it to the current class.
IPC_MESSAGE_FORWARD(ViewHostMsg_MyMessage, some_object_pointer, SomeObject::OnMyMessage)
- IPC_MESSAGE_HANDLER_GENERIC: This allows you to write your own code, but you have to unpack the parameters from the message yourself:

IPC_MESSAGE_HANDLER_GENERIC(ViewHostMsg_MyMessage, printf("Hello, world, I got the message."))
###Security considerations

Security bugs in IPC can have [nasty consequences](http://blog.chromium.org/2012/05/tale-of-two-pwnies-part-1.html) (file theft, sandbox escapes, remote code execution). Check out our [security for IPC](https://www.chromium.org/Home/chromium-security/education/security-tips-for-ipc) document for tips on how to avoid common pitfalls.

##Channels

IPC::Channel (defined in ipc/ipc_channel.h) defines the methods for communicating across pipes. IPC::SyncChannel provides additional capabilities for synchronously waiting for responses to some messages (the renderer processes use this as described below in the "Synchronous messages" section, but the browser process never does).

Channels are not thread safe. We often want to send messages using a channel on another thread. For example, when the UI thread wants to send a message, it must go through the I/O thread. For this, we use a IPC::ChannelProxy. It has a similar API as the regular channel object, but proxies messages to another thread for sending them, and proxies messages back to the original thread when receiving them. It allows your object (typically on the UI thread) to install a IPC::ChannelProxy::Listener on the channel thread (typically the I/O thread) to filter out some messages from getting proxied over. We use this for resource requests and other requests that can be handled directly on the I/O thread. RenderProcessHost installs a RenderMessageFilter object that does this filtering.

##Synchronous messages

Some messages should be synchronous from the renderer's perspective. This happens mostly when there is a WebKit call to us that is supposed to return something, but that we must do in the browser. Examples of this type of messages are spell-checking and getting the cookies for JavaScript. Synchronous browser-to-renderer IPC is disallowed to prevent blocking the user-interface on a potentially flaky renderer.

**Danger**: Do not handle any synchronous messages in the UI thread! You must handle them only in the I/O thread. Otherwise, the application might deadlock because plug-ins require synchronous painting from the UI thread, and these will be blocked when the renderer is waiting for synchronous messages from the browser.

###Declaring synchronous messages

Synchronous messages are declared using the IPC_SYNC_MESSAGE_* macros. These macros have input and return parameters (non-synchronous messages lack the concept of return parameters). For a control function which takes two input parameters and returns one parameter, you would append 2_1 to the macro name to get:

IPC_SYNC_MESSAGE_CONTROL2_1(SomeMessage,  // Message name
                            GURL, //input_param1
                            int, //input_param2
                            std::string); //result
Likewise, you can also have messages that are routed to the view in which case you would replace "control" with "routed" to get IPC_SYNC_MESSAGE_ROUTED2_1. You can also have 0 input or return parameters. Having no return parameters is used when the renderer must wait for the browser to do something, but needs no results. We use this for certain printing and clipboard operations.

###Issuing synchronous messages

When the WebKit thread issues a synchronous IPC request, the request object (derived from IPC::SyncMessage) is dispatched to the main thread on the renderer through a IPC::SyncChannel object (the same one is also used to send all asynchronous messages). The SyncChannel will block the calling thread when it receives a synchronous message, and will only unblock it when the reply is received.

While the WebKit thread is waiting for the synchronous reply, the main thread is still receiving messages from the browser process. These messages will be added to the queue of the WebKit thread for processing when it wakes up. When the synchronous message reply is received, the thread will be un-blocked. Note that this means that the synchronous message reply can be processed out-of-order.

Synchronous messages are sent the same way normal messages are, with output parameters being given to the constructor. For example:
```c++
const GURL input_param("http://www.google.com/");
std::string result;
content::RenderThread::Get()->Send(new MyMessage(input_param, &result));
printf("The result is %s\n", result.c_str());
```
###Handling synchronous messages

Synchronous messages and asynchronous messages use the same IPC_MESSAGE_HANDLER, etc. macros for dispatching the message. The handler function for the message will have the same signature as the message constructor, and the function will simply write the output to the output parameter. For the above message you would add

IPC_MESSAGE_HANDLER(MyMessage, OnMyMessage)
to the OnMessageReceived function, and write:
```c++
void RenderProcessHost::OnMyMessage(GURL input_param, std::string* result) {
  *result = input_param.spec() + " is not available";
}
```
###Converting message type to a message name

If you get a crash and you have the message type you can convert this to a message name. The message type will be 32-bit value, the high 16-bits are the class and the low 16-bits are the id. The class is based on the enums in ipc/ipc_message_start.h, the id is based on the line number in the file that defines the message. This means that you need to get the exact revision of Chromium in order to accurately get the message name.

Example of this in [554011](https://crbug.com/554011) was 0x1c0098 at Chromium revision [ad0950c1ac32ef02b0b0133ebac2a0fa4771cf20](https://crrev.com/ad0950c1ac32ef02b0b0133ebac2a0fa4771cf20). That's class 0x1c which is line [40](https://chromium.googlesource.com/chromium/src/+/ad0950c1ac32ef02b0b0133ebac2a0fa4771cf20/ipc/ipc_message_start.h#40) which matches ChildProcessMsgStart. ChildProcessMsgStart messages are in content/common/child_process_messages.h and the IPC will be on line 0x98 or line 152 which is ChildProcessHostMsg_ChildHistogramData.

This technique is particularly useful if you are dealing with crashes caused by content::RenderProcessHostImpl::OnBadMessageReceived