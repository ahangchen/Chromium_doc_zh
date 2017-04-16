# 跨进程通信 (IPC)
## 概览

Chromium有一个[多进程架构](../Start_Here_Background_Reading/Multi-process_Architecture.md)，这意味着我们有许多需要互相交流的进程。我们的主要跨进程交流元素是命名管道。在Linux和OS X上，我们使用socketpair()。每个渲染器进程可以分配到一个命名管道来跟浏览器进程交流。这些管道是用异步方式使用的，确保没有哪个端会等待另一个端。

想要得到如何编写安全的IPC端点的知识，请查看[IPC安全要点](https://www.chromium.org/Home/chromium-security/education/security-tips-for-ipc).

### 浏览器中IPC

在浏览器中，与渲染器的交流是通过一个独立的I/O线程完成的。来自或者去往view的消息需要使用一个ChannelProxy代理到主线程。这种方案的优点是，资源请求（比如网页等），这种最经常且极其关注性能的消息，可以整个的在I/O线程中处理，不会阻塞用户界面。这些通过使用Channel::MessageFilter（由RenderProcessHost插入channel）来完成。这个过滤器运行在I/O线程里，拦截资源请求信息，将它们直接转发到资源分发主机。查看[多进程资源加载](General_Architecture/Multi-process/Multi-process_Resource_Loading.md)获取更多关于资源加载的信息。

### 渲染器中的IPC

每个渲染器也有一个线程管理交流（在这个例子里，是主线程），而大多数渲染和大多数处理发生在另一个线程里（查看多进程架构的那个图表）。大多数消息通过主渲染线程从浏览器发送给WebKit线程，反之亦然。这个额外的线程是用于支持同步的渲染器到浏览器的消息（参考下面的“同步消息”）。



## 消息

### 消息的类型

我们有两种基本的消息类型：”路由“和”控制“。控制消息由创建管道的类处理，有时候这个类允许其他人通过一个MessageRouter对象接收消息，其他监听器可以通过这个对象注册和接收有着唯一管道id的消息。

例如，渲染时，控制消息没有消息指定目标view，会被RenderProcess（渲染器）或者RenderProcessHost（浏览器）处理。来自资源的请求或者修改剪贴板的请求是没有目标view的，所以是控制消息。一个路由消息的例子是，要求一个view绘制一个区域的消息。

路由消息曾经被用于从指定的RenderViewHost获取消息。然而，技术上，任何类可以通过使用RenderProcessHost::GetNextRoutingID接收路由消息，并用RenderProcessHost::AddRoute注册它自己这个消息。现在，RenderFrameHost和RenderViewHost有了他们自己的路由ID了。

消息是否是独立类型在于，消息是从浏览器发送到渲染器，还是从渲染器到浏览器。从浏览器到渲染器的被称为View消息，因为它们被发送给RenderViewHost。从渲染器发送到浏览器的消息叫做ViewHost消息，因为他们被发送给RenderViewHost。你会注意到render_messages_internal.h里定义的消息被分为两类。

插件也有独立的进程。像渲染消息那样，PluginProcess消息（从浏览器发送到插件进程）和PluginProcessHost消息（从插件进程发送到浏览器）。这些消息都定义在plugin_messages_internal.h里。自动化消息（用于控制浏览器做UI测试）通过相同的方式完成。

### 声明消息

特殊的宏用于声明消息。渲染器和浏览器间发送的消息都声明在render\_messages\_internal.h里。有两个部分，一个是发送到渲染器的View消息，一个是发送到浏览器的ViewHost消息。

如果要声明一个从渲染器发送到浏览器（一个ViewHost消息）的消息，并且指定一个view（路由）包含一个url和一个整数作为参数，这样写：

```c++
IPC_MESSAGE_ROUTED2(ViewHostMsg_MyMessage, GURL, int)
```

如果要声明一个从浏览器发往渲染器的控制消息（一个View消息），并且不指定目标view（控制），不包含参数，这样写：

```c++
IPC_MESSAGE_CONTROL0(ViewMsg_MyMessage)
```
#### 包装数据

参数通过ParamTraits模板序列化或者反序列化到消息体中。这种模板的具体化在ipc_message_utils.h中提供给大多数常见的类型。如果你定义了你自己的类型，你也需要为它定义你自己的ParamTraits具体形式。

有时候，一条消息有太多的值了，没法合理得放到消息里。这种情况下，我们定义一个独立的结构来存放这些值。例如，对于FrameMsg_Navigate消息，在[navigation_params.h](https://code.google.com/p/chromium/codesearch/#chromium/src/content/common/navigation_params.h)中，定义了CommonNavigationParams结构。[frame_messages.h](https://code.google.com/p/chromium/codesearch/#chromium/src/content/common/frame_messages.h)用[IPC_STRUCT_TRAITS](https://code.google.com/p/chromium/codesearch/#chromium/src/ipc/param_traits_macros.h)类型的宏定义了这个结构的具体ParamTraits。


### 发送消息

你通过“channel（通道）”发送消息（往下看）。在浏览器里，RenderProcessHost包含了用于从浏览器UI线程发送消息到渲染器的channel。为了方便，RenderWidgetHost（RenderViewHost的基类）提供了一个Send函数。

消息由指针发送，并将在它们分发后由IPC层删除。因此，一旦你发现合适的Send函数，尽管带着一条新消息去调用它：
```c++
Send(new ViewMsg_StopFinding(routing_id_));
```

注意，你必须按顺序指定路由ID，让消息能够路由到接收端正确的View/ViewHost。RenderWidgetHost(RenderViewHost的基类)和RenderWidget（RenderViewHost的基类）都有GetRoutingID()成员函数给你使用。


### 处理消息

消息由对IPC::Listener的实现来处理，其中最重要的函数是OnMessageReceived。我们有大量的宏来简化这个函数中的消息处理，这个最好可以用例子来阐述：


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

你也可以使用IPC_DEFINE_MESSAGE_MAP来实现自己的函数定义。在这个例子里，不要指定消息变量名，它会在给定的类上声明一个OnMessageReceived函数，并实现之。

其他宏：

- IPC_MESSAGE_FORWARD:这与IPC_MESSAGE_HANDLER相同，但你可以指定你自己的类来作为消息发送的目的地，而非发送给当前类。
```c++
IPC_MESSAGE_FORWARD(ViewHostMsg_MyMessage, some_object_pointer, SomeObject::OnMyMessage)
```

- IPC_MESSAGE_HANDLER_GENERIC:这允许你编写自己的代码，但你必须自己从消息中解包出参数。
```c++
IPC_MESSAGE_HANDLER_GENERIC(ViewHostMsg_MyMessage, printf("Hello, world, I got the message."))
```

### 安全考虑

IPC中的安全漏洞有着[严重的后果](http://blog.chromium.org/2012/05/tale-of-two-pwnies-part-1.html)(文件盗取，沙箱逃逸，远程代码执行)，查看我们的[IPC安全文档](https://www.chromium.org/Home/chromium-security/education/security-tips-for-ipc)以获取如何避免常见陷阱的一些提示。

## 通道

IPC::Channel()（定义在ipc/ipc_channel.h里）定义了通过管道交流的方法。IPC::SyncChannel提供了额外的功能用于同步等待一些消息的响应（正如下面的“同步消息”描述的，渲染器进程使用了这个特性，但浏览器进程不会这样做）。

通道不是线程安全的，我们通常希望用通道在另一个线程里发送消息。例如，当UI线程希望发送消息时，它必须通过I/O线程。为此，我们使用IPC::ChannelProxy。它有着与正常通道对象类似的API，但它把消息代理到另一个线程去发送，而在收到这些消息时，把消息代理回原来的线程。这允许你的对象（通常是在UI线程中）在通道线程（通常是在I/O线程中）安装一个IPC::ChannelProxy::Listener，以此从代理的消息中过滤掉一些消息。我们使用这个特性去做资源请求以及其他可以直接在I/O线程处理的请求。RenderProcessHost安装一个RenderMessageFilter对象执行这种过滤。


## 同步消息
有些消息应该从渲染器的角度来同步。这大多数时候发生在，有一个支持返回值的WebKit调用，但我们必须在浏览器中执行这个调用。这种消息的例子是拼写检查以及在javaScript中获取cookie。同步浏览器到渲染器的IPC是不允许的，以此避免在一个潜在的片段渲染器中阻塞用户界面。

**警告**: 不要在UI线程处理任何同步消息！你必须在I/O线程中处理他们。否则，应用程序可能因为插件等待UI线程的同步绘制而陷入死锁，而渲染器等待浏览器同步消息时也会有一些阻塞。


### 声明同步消息

同步消息用IPC\_SYNC\_MESSAGE\_\*这样的宏来声明。这些宏有输入，也有返回值()(非同步消息没有返回参数的概念)。对于一个有着两个输入参数和一个返回参数的控制函数，你应该在宏的名字中插入“2\_1”：
```c++
IPC_SYNC_MESSAGE_CONTROL2_1(SomeMessage,  // Message name
                            GURL, //input_param1
                            int, //input_param2
                            std::string); //result
```

类似的，你也可以让消息路由到view，这种情况下你需要把“CONTROL”换成“ROUTED”，得到IPC\_SYNC\_MESSAGE\_\*。你也可以没有输入或返回参数。没有返回参数常用于渲染器必须等待浏览器完成某些操作但不需要结果时。我们在某些打印和剪贴板操作使用这种特性。


### 分发同步消息

当WebKit线程分发出一个同步IPC请求时，请求对象（继承自IPC::SyncMessage）会在渲染器中通过IPC::SyncChannel对象分发给主线程。所有同步的消息也是通过它发送的。同步通道在接收到同步消息时，会阻塞调用线程，只有当收到回复时，才会解除阻塞。

在WebKit线程等待同步请求时，主线程仍然会从浏览器进程接收消息。这些消息会添加到WebKit线程里，等到WebKit线程被唤醒时处理它们。当同步消息回复被接收时，这个线程会解除阻塞。注意这意味着同步消息回复可以不按顺序处理。

同步消息和正常的消息用同样的方式，带着赋予构造器的输出参数发送出去，例如：

```c++
const GURL input_param("http://www.google.com/");
std::string result;
content::RenderThread::Get()->Send(new MyMessage(input_param, &result));
printf("The result is %s\n", result.c_str());
```
### 处理同步消息

同步消息和异步消息使用相同的IPC\_MESSAGE\_HANDLER等宏来分发消息。消息处理函数与消息构造器有着相同的函数签名，这个函数会简单把输出写到输出参数中。在上面的消息里，你可以添加：
```c++
IPC_MESSAGE_HANDLER(MyMessage, OnMyMessage)
```
到OnMessageReceived函数，然后这样写：
```c++
void RenderProcessHost::OnMyMessage(GURL input_param, std::string* result) {
  *result = input_param.spec() + " is not available";
}
```
### 转换消息类型为消息名

如果运行崩溃了，并且此时你有消息的类型，你可以把它转为消息名。这种消息类型是一个32位值，高16位是类，低16位是ID。类基于ipc/ipc\_message\_start.h中的枚举值，id基于定义消息的文件中的行号。这意味着你需要获取准确的Chromium版本以获取消息名。

一个[554011](https://crbug.com/554011)中的例子是Chromium [ad0950c1ac32ef02b0b0133ebac2a0fa4771cf20](https://crrev.com/ad0950c1ac32ef02b0b0133ebac2a0fa4771cf20) 版0x1c0098中。类0x1c，意味着行[40](https://chromium.googlesource.com/chromium/src/+/ad0950c1ac32ef02b0b0133ebac2a0fa4771cf20/ipc/ipc_message_start.h#40)，匹配ChildProcessMsgStart。ChildProcessMsgStart消息在content/common/child_process_messages.h中，而0x98行，即152行，对应的IPC是ChildProcessHostMsg\_ChildHistogramData.

当你在处理content::RenderProcessHostImpl::OnBadMessageReceived导致的crash时，这项技术非常有用。
