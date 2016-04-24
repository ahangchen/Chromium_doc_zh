#线程

##概览

Chromium是一个极其多线程的产品。我们努力让UI尽可能快速响应，这意味着任何阻塞I/O或者其他昂贵操作不能阻塞UI线程。我们的做法是在线程间传递消息作为交流的方式。我们不鼓励锁和线程安全对象。相反的，对象仅存在与单个线程中，我们只为通信而在线程间传递消息，我们会在大多数跨进程请求间使用回调接口（由消息传递实现）。

Thread对象定义于base/threading/thread.h中。通常你可能会使用下面描述的已有线程之一而非重新构建线程。我们已经有了很多难以追踪的线程。每个线程有一个消息循环（查看base/message_loop/message_loop.h），消息循环处理这个线程的消息。你可以使用Thread.message_loop()函数获取一个线程对应的消息循环。更多关于消息循环的内容可以在这里查看[Anatomy of Chromium MessageLoop](https://docs.google.com/document/d/1_pJUHO3f3VyRSQjEhKVvUU7NzCyuTCQshZvbWeQiCXU/edit#).


##已有线程

大多数线程由BrowserProcess对象管理，它是主“浏览器”进程的服务管理器。默认情况下，所有的事情都发生在UI线程中。我们已经把某些类的处理过程放到了其他一些线程里。下面这些线程有getter接口：

* **ui_thread**: 应用从这个主线程启动
* **io_thread**: 某种程度上讲，这个线程起错名字了。它是一个分发线程，它处理浏览器进程和其他所有子进程之间的交流。它也是所有资源请求（页面加载的）分发的起点（查看[多进程架构](../Start_Here_Background_Reading/Multi-process_Architecture.md)）。
* **file_thread**: 一个用于文件操作的普通线程。当你想要做阻塞文件系统的操作（例如，为某种文件类型请求icon，或者向磁盘写下载文件），分配给这个线程。
* **db_thread**: 用于数据库操作的线程。例如，cookie服务在这个线程上执行sqlite操作。注意，历史记录数据库还不会使用这个线程。
* **safe_browsing_thread**

几个组件有它们自己的线程：

* **History**: 历史记录服务有它自己的线程。这可能会和上面的db_thread合并。然而我们需要保证这会按照正确的顺序发生 -- 例如，cookie在历史记录前会仙贝加载，因为首次加载需要cookie，历史记录初始化需要很长时间，会阻塞cookie的加载。
* **Proxy service**: 查看net/http/http_proxy_service.cc.
* **Automation proxy**: 这个线程用于和驱动应用的UI测试程序交流。

##保持浏览器积极响应

正如上面所暗示的，我们在UI线程里避免任何阻塞I/O，以保持UI积极响应。另一个不太明显的点是，我们也需要避免io_thread里执行阻塞I/O。因为如果我们因昂贵的操作阻塞了这个线程，比如磁盘访问，那么IPC信息不会得到处理，结果就是用户不能与页面进行交互。注意异步/平行 IO是可以的。

另一个需要注意的事情是，不要在一个线程里阻塞另一个线程。锁只能用于交换多线程访问的共享数据。如果一个线程基于昂贵的计算或者通过磁盘访问而更新，那么应当在不持有锁的情况下完成缓慢的工作。只有在结果可用后，锁才应该用于交换新数据。一个例子是，在PluginList::LoadPlugins (src/content/common/plugin_list.cc)中。如果你必须使用锁，[这里](https://www.chromium.org/developers/lock-and-condition-variable)有一些最佳实践以及一些需要避开的陷阱。

为了编写不阻塞的代码，许多Chromium中的API是异步的。通常这意味着他们需要在一个特殊的线程里执行，并通过自定义的装饰接口返回结果，或者他们会在请求操作完成后调用base::Callback<>对象。在具体线程执行的工作会在下面的PostTask章节介绍。


##把事情放到其他线程去

###base::Callback<>, 异步APIs, 和Currying

base::Callback<> (查看[callback.h的文档](http://src.chromium.org/viewvc/chrome/trunk/src/base/callback.h?content-type=text%2Fplain)) 是有着一个Run()方法的模板类。它由对base::Bind的调用来创建。异步API通常将base::Callback<>作为一种异步返回操作结果的方式。这是一个假想的文件阅读API的例子。

```c++
void ReadToString(const std::string& filename, const base::Callback<void(const std::string&)>& on_read);
```
```c++
void DisplayString(const std::string& result) {
  LOG(INFO) << result;
}
```
```c++
void SomeFunc(const std::string& file) {
  ReadToString(file, base::Bind(&DisplayString));
};
```
在上面的例子中，base::Bind拿到&DisplayString的函数指针然后将它传给base::Callback&lt;void(const std::string& result)&gt;。生成的base::Callback<>的类型依赖于传入参数。为什么不直接传入函数指针呢？原因是base::Bind允许调用者适配功能接口或者通过Currying(http://en.wikipedia.org/wiki/Currying)绑定具体的上下文。例如，如果我们有一个工具函数DisplayStringWithPrefix，它接受一个有着前缀的具体参数，我们使用base::Bind以适配接口，如下所示：


```c++
void DisplayStringWithPrefix(const std::string& prefix, const std::string& result) {
  LOG(INFO) << prefix << result;
}
```
```c++
void AnotherFunc(const std::string& file) {
  ReadToString(file, base::Bind(&DisplayStringWithPrefix, "MyPrefix: "));
};
```
这可以作为创建适配器，在一个小的类中将前缀作为成员变量而持有，的替代方案。也要注意“MyPrefix: ”参数事实上是一个const char\*，而DisplayStringWithPrefix需要的其实是const std::string&。正如常见的函数分配那样，base::Bind，可能的话，会进行强制参数类型转化。查看下面的“base::Bind()如何处理参数”以获取关于参数存储，复制，以及对引用的特殊处理的更多细节。


###PostTask

分发线程的最底层是使用MessageLoop.PostTask和MessageLoop.PostDelayedTask（查看base/message_loop/message_loop.h）。PostTask会在一个特别的线程上进行一个任务调度。这个任务定义为一个base::Closure，这是base::Callback&lt;void(void)&gt;的一个子类型。PostDelayedTask会在一个特殊线程里，延迟发起一个任务。这个任务由base::Closure类表示，它包含一个Run()方法，并在base::Bind()被调用时创建。处理任务时，消息循环最终会调用base::CLosure的Run函数，然后丢掉对任务对象的引用。PostTask和PostDelayedTask都会接收一个tracked_objects::Location参数，用于轻量级调试（挂起的和完成的任务的计数和初始分析可以在调试构建版本中通过about:objects地址进行监控）。通常FROM_HERE宏的值适合赋给这个参数的。

注意新的任务运行在消息循环队列里，任何指定的延迟受操作系统定时器策略影响。这意味着在Windows平台，非常小的超时（10毫秒内）很可能是不会发生的（超时时间会更长）。在PostDelayedTask里将超时时间设置为0也可以用于在当前线程里，当前进程返回消息队列之后的某个时候。当前线程中这样的一种持续可以用于确保其他时间敏感的任务不会在这个线程上进入饥饿状态。

下面是一个为一个功能创建一个任务然后在另一个线程上执行这个任务的例子（在这个例子里，在文件线程里）：

```c++
void WriteToFile(const std::string& filename, const std::string& data);
BrowserThread::PostTask(BrowserThread::FILE, FROM_HERE,
                        base::Bind(&WriteToFile, "foo.txt", "hello world!"));
```
你应该总使用BrowserThread在线程间提交任务。永远不要缓存MessageLoop指针，因为它会导致一些bug，比如当你还持有指针时，它们被删掉了。更多信息可以在[这里](https://www.chromium.org/developers/design-documents/threading/suble-threading-bugs-and-patterns-to-avoid-them)找到。

###base::Bind()和类方法

base::Bind() API也支持调用类方法。语法与在一个函数里调用base::Bind()类似，除了第一个参数必须是这个方法所属的对象。默认情况下，PostTask使用的对象必须是一个线程安全引用计数对象。引用计数保证了另一个线程调用的对象必须在线程完成前保活。

```c++
class MyObject : public base::RefCountedThreadSafe<MyObject> {
 public:
  void DoSomething(const std::string16& name) {
    thread_->message_loop()->PostTask(
       FROM_HERE, base::Bind(&MyObject::DoSomethingOnAnotherThread, this, name));
  }

  void DoSomethingOnAnotherThread(const std::string16& name) {
    ...
  }
 private:
  // Always good form to make the destructor private so that only RefCountedThreadSafe can access it.
  // This avoids bugs with double deletes.
  friend class base::RefCountedThreadSafe<MyObject>;

  ~MyObject();
  Thread* thread_;
};
```
如果你有外部同步结构，而且它能保证对象在任务正在等待执行期间一直保活，你就可以在调用base::Bind()时用base::Unratained()包装这个对象指针，以关闭引用计数。这也允许在非引用计数的类上使用base::Bind()。但请小心处理这种情况！

###base::Bind()怎么处理参数

传给base::Bind()的参数会被复制到一个内部InvokerStorage结构对象（定义在base/bind_internal.h中）。当这个函数最终执行时，它会查看参数的这些副本。如果你的目标函数或者方法持有常量引用时，这是很重要的；这些引用会变成一份参数的副本。如果你需要原始参数的引用，你可以用base::ConstRef()包装参数。小心使用这个函数，因为如果引用的目标不能保证在任务执行过程中一直存活的话，这会很危险。尤其是，为栈中的变量调用base::ConstRef()几乎一定是不安全的，除非你可以保证栈帧不会在异步任务完成前无效化。

有时候，你会想要传递引用技术对象作为参数（请确保使用RefCountedThreadSafe，并且为这些对象做为基类的纯引用计数）。为了保证对象在整个请求期间都能存活，base::Bind()生成的Closure必须持有这个对象的引用。这可以通过将scoped_refptr作为参数类型传递，或者用make_scoped_refptr()包装裸指针来完成:
```c++
class SomeParamObject : public base::RefCountedThreadSafe<SomeParamObject> {
 ...
};

class MyObject : public base::RefCountedThreadSafe<MyObject> {
 public:
  void DoSomething() {
    scoped_refptr<SomeParamObject> param(new SomeParamObject);
    thread_->message_loop()->PostTask(FROM_HERE
       base::Bind(&MyObject::DoSomethingOnAnotherThread, this, param));
  }
  void DoSomething2() {
    SomeParamObject* param = new SomeParamObject;
    thread_->message_loop()->PostTask(FROM_HERE
       base::Bind(&MyObject::DoSomethingOnAnotherThread, this, 
                         make_scoped_refptr(param)));
  }
  // Note how this takes a raw pointer. The important part is that
  // base::Bind() was passed a scoped_refptr; using a scoped_refptr
  // here would result in an extra AddRef()/Release() pair.
  void DoSomethingOnAnotherThread(SomeParamObject* param) {
    ...
  }
};
```
如果你想要不持有引用而传递对象，就要用base::Unretained()包装参数。再一次，使用这个函数意味着需要有对对象的生命周期的外部保证，所以请小心使用！

如果你的对象有一个特殊的析构函数，它需要在特殊的线程运行，你可以使用下面的特性。因为计时可能导致任务的代码展开栈前，任务就完成了，所以这是必要的：

```c++
class MyObject : public base::RefCountedThreadSafe<MyObject, BrowserThread::DeleteOnIOThread> {
```
##撤销回调

撤销任务主要有两个原因（以回调的形式）：

* 你希望在之后对对象做一些事情，但在你的回调运行时，你的对象可能被销毁。
* 当输入改变时（例如，用户输入），旧的任务会变得不必要。出于性能考虑，你应该取消它们。

查看下面不同的方式取消任务：

###关于撤销任务的重要提示

撤销一个持有参数的任务是很危险的。查看下面的例子（这里例子使用base::WeakPtr以执行撤销操作，但问题适用于其他情景）。

```c++
class MyClass {
 public:
  // Owns |p|.
  void DoSomething(AnotherClass* p) {
    ...
  }
  WeakPtr<MyClass> AsWeakPtr() {
    return weak_factory_.GetWeakPtr();
  }
 private:
  base::WeakPtrFactory<MyClass> weak_factory_;
};
```
      
...
```c++
Closure cancelable_closure = Bind(&MyClass::DoSomething, object->AsWeakPtr(), p);
Callback<void(AnotherClass*)> cancelable_callback = Bind(&MyClass::DoSomething, object->AsWeakPtr());
```
      
...
```c++
void FunctionRunLater(const Closure& cancelable_closure,
                      const Callback<void(AnotherClass*)>& cancelable_callback) {   
  // Leak memory!
  cancelable_closure.Run();
  cancelable_callback.Run(p);
}
```
在FunctionRunLater中，当对象已经被销毁时，Run()调用会泄露p。使用scoped_ptr可以修复这个bug。
```c++
class MyClass {
 public:
  void DoSomething(scoped_ptr<AnotherClass> p) {
    ...
  }
  ...
};
```
###base::WeakPtr和撤销[非线程安全]

你可以使用base::WeakPtr和base::WeakPtrFactory(在base/memory/weak_ptr.h)以确保任何调用不会超过它们调用的对象的生命周期，而不执行引用计数。base::Bind机制对base::WeakPtr有特殊的理解，会在base::WeakPtr已经失效的情况下终止任务的执行。base::WeakPtrFactory对象可以用于生成base::WeakPtr实例，这些实例被工厂对象引用。当工厂被销毁时，所有的base::WeakPtr会设置它们内部的“invalidated”标志位，这些标志位会使得与其绑定的任何任务不被分发。通过将工厂作为被分发的对象的成员，可以实现自动撤销。

注意：这只在任务传递到相同的线程时才能生效。当前没有对于分发到其他线程的任务能够生效的普适解决方案。查看下一节，关于CancelableTaskTracker，了解其他解决方案。
```c++
class MyObject {
 public:
  MyObject() : weak_factory_(this) {}

  void DoSomething() {
    const int kDelayMS = 100;
    MessageLoop::current()->PostDelayedTask(FROM_HERE,
        base::Bind(&MyObject::DoSomethingLater, weak_factory_.GetWeakPtr()),
        kDelayMS);
  }

  void DoSomethingLater() {
    ...
  }

 private:
  base::WeakPtrFactory<MyObject> weak_factory_;
};
```
###CancelableTaskTracker
当base::WeakPtr在撤销任务时非常有效，它不是线程安全的，因此不能被用于取消运行在其他线程的任务。有时候会有关注性能的需求。例如，我们需要在用户改变输入文本时，撤销在DB线程的数据库查询任务。在这种情况下，CancelableTaskTracker比较合适。



With CancelableTaskTracker you can cancel a single task with returned TaskId. This is another reason to use CancelableTaskTracker instead of base::WeakPtr, even in a single thread context.

CancelableTaskTracker has 2 Post methods doing the same thing as the ones in base::TaskRunner, with additional cancellation support. 
```c++
class UserInputHandler : public base::RefCountedThreadSafe<UserInputHandler> {
  // Runs on UI thread.
  void OnUserInput(Input input) {
    CancelPreviousTask();
    DBResult* result = new DBResult();
    task_id_ = tracker_->PostTaskAndReply(
        BrowserThread::GetMessageLoopProxyForThread(BrowserThread::DB).get(),
        FROM_HERE,
        base::Bind(&LookupHistoryOnDBThread, this, input, result),
        base::Bind(&ShowHistoryOnUIThread, this, base::Owned(result)));
  }

  void CancelPreviousTask() {
    tracker_->TryCancel(task_id_);
  }

  ...

 private:
  CancelableTaskTracker tracker_;  // Cancels all pending tasks while destruction.
  CancelableTaskTracker::TaskId task_id_;
  ...
};
```
Since task runs on other threads, there's no guarantee it can be successfully canceled.

When TryCancel() is called:
* If neither task nor reply has started running, both will be canceled.
* If task is already running or has finished running, reply will be canceled.
* If reply is running or has finished running, cancelation is a noop.

Like base::WeakPtrFactory, CancelableTaskTracker will cancel all tasks on destruction.

###Cancelable request (DEPRECATED)

Note. Cancelable request is deprecated. Please do not use it in new code. For canceling tasks running on the same thread, use WeakPtr. For canceling tasks running on a different thread, use CancelableTaskTracker.

A cancelable request makes it easier to make requests to another thread with that thread returning some data to you asynchronously. Like the revokable store system, it uses objects that track whether the originating object is alive. When the calling object is deleted, the request will be canceled to prevent invalid callbacks.
Like the revokable store system, a user of a cancelable request has an object (here, called a "Consumer") that tracks whether it is alive and will auto-cancel any outstanding requests on deleting.
```c++
class MyClass {
  void MakeRequest() {
    frontend_service->StartRequest(some_input1, some_input2, this,
        // Use base::Unretained(this) if this may cause a refcount cycle.
        base::Bind(&MyClass:RequestComplete, this));  
  }
  void RequestComplete(int status) {
    ...
  }

 private:
  CancelableRequestConsumer consumer_;
};
```
Note that the MyClass::RequestComplete, is bounded with base::Unretained(this) here.

The consumer also allows you to associate extra data with a request. Use CancelableRequestConsumer which will allow you to associate arbitrary data with the handle returned by the provider service when you invoke the request. The data will be automatically destroyed when the request is canceled.

A service handling requests inherits from CancelableRequestProvider. This object provides methods for canceling in-flight requests, and will work with the consumers to make sure everything is cleaned up properly on cancel. This frontend service just tracks the request and sends it to a backend service on another thread for actual processing. It would look like this:
```c++
class FrontendService : public CancelableRequestProvider {
  typedef base::Callback<void(int)> RequestCallbackType;

  Handle StartRequest(int some_input1, int some_input2,
      CallbackConsumer* consumer,
      const RequestCallbackType& callback) {
    scoped_refptr< CancelableRequest<FrontendService::RequestCallbackType> >
        request(new CancelableRequest(callback));
    AddRequest(request, consumer);

    // Send the parameters and the request to the backend thread.
    backend_thread_->PostTask(FROM_HERE,
        base::Bind(&BackendService::DoRequest, backend_, request,
                   some_input1, some_input2), 0);    
    // The handle will have been set by AddRequest.
    return request->handle();
  }
};
```
The backend service runs on another thread. It does processing and forwards the result back to the original caller. It would look like this:
```c++
class BackendService : public base::RefCountedThreadSafe<BackendService> {
  void DoRequest(
      scoped_refptr< CancelableRequest<FrontendService::RequestCallbackType> >
          request,
      int some_input1, int some_input2) {
    if (request->canceled())
      return;

    ... do your processing ...

    // Execute ForwardResult() like you would do Run() on the base::Callback<>.
    request->ForwardResult(return_value);
  }
};
```