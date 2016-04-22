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

注意新的任务运行在消息循环队列里，任何指定的延迟受操作系统定时器策略影响。这意味着在Windows平台，非常小的超时（10毫秒内）很可能是不会发生的（超时时间会更长）
Note that new tasks go on the message loop's queue, and any delay that is specified is subject to the operating system's timer resolutions. This means that under Windows, very small timeouts (under 10ms) will likely not be honored (and will be longer). Using a timeout of 0 in PostDelayedTask is equivalent to calling PostTask, and adds no delay beyond queuing delay. PostTask is also used to do something on the current thread "sometime after the current processing returns to the message loop." Such a continuation on the current thread can be used to assure that other time critical tasks are not starved on this thread.

The following is an example of a creating a task for a function and posting it to another thread (in this example, the file thread):
```c++
void WriteToFile(const std::string& filename, const std::string& data);
BrowserThread::PostTask(BrowserThread::FILE, FROM_HERE,
                        base::Bind(&WriteToFile, "foo.txt", "hello world!"));
```
You should always use BrowserThread to post tasks between threads.  Never cache MessageLoop pointers as it can cause bugs such as the pointers being deleted while you're still holding on to them.  More information can be found [here](https://www.chromium.org/developers/design-documents/threading/suble-threading-bugs-and-patterns-to-avoid-them).

###base::Bind() and class methods.

The base::Bind() API also supports invoking class methods as well.  The syntax is very similar to calling base::Bind() on a function, except the first argument should be the object the method belongs to. By default, the object that PostTask uses must be a thread-safe reference-counted object. Reference counting ensures that the object invoked on another thread will stay alive until the task completes.
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
If you have external synchronization structures that can completely insure that an object will always be alive while the task is waiting to execute, you can wrap the object pointer with base::Unretained() when calling base::Bind() to disable the refcounting.  This will also allow using base::Bind() on classes that are not refcounted.  Be careful when doing this!
###How arguments are handled by base::Bind().

The arguments given to base::Bind() are copied into an internal InvokerStorage structure object (defined in base/bind_internal.h). When the function is finally executed, it will see copies of the arguments.  This is important if your target function or method takes a const reference; the reference will be to a copy of the argument.  If you need a reference to the original argument, you can wrap the argument with base::ConstRef().  Use this carefully as it is likely dangerous if target of the reference cannot be guaranteed to live past when the task is executed.  In particular, it is almost never safe to use base::ConstRef() to a variable on the stack unless you can guarantee the stack frame will not be invalidated until the asynchronous task finishes.

Sometimes, you will want to pass reference-counted objects as parameters (be sure to use RefCountedThreadSafe and not plain RefCounted as the base class for these objects). To ensure that the object lives throughout the entire request, the Closure generated by base::Bind must keep a reference to it. This can be done by passing scoped_refptr as the parameter type, or by wrapping the raw pointer with make_scoped_refptr():
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
If you  want to pass the object without taking a reference on it, wrap the argument with base::Unretained(). Again, using this means there are external guarantees on the lifetime of the object, so tread carefully!

If your object has a non-trivial destructor that needs to run on a specific thread, you can use the following trait. This is needed since timing races could lead to a task completing execution before the code that posted it has unwound the stack.
```c++
class MyObject : public base::RefCountedThreadSafe<MyObject, BrowserThread::DeleteOnIOThread> {
```
##Callback cancellation

There are 2 major reasons to cancel a task (in the form of a Callback):
* You want to do something later on your object, but at the time your callback runs, your object may have been destroyed.
* When input changes (e.g. user input), old tasks become unnecessary. For performance consideration, you should cancel them.
See following about different approaches for cancellation.
###Important notes about cancellation

It 's dangerous to cancel a task with owned parameters. See following example. (The example uses base::WeakPtr for cancellation, but the problem applies to all approaches).
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
```
      
  // Leak memory!
  cancelable_closure.Run();
  cancelable_callback.Run(p);
}
```
In FunctionRunLater, both Run() calls will leak p when object is already destructed. Using scoped_ptr can fix the bug.
```c++
class MyClass {
 public:
  void DoSomething(scoped_ptr<AnotherClass> p) {
    ...
  }
  ...
};
```
###base::WeakPtr and Cancellation [NOT THREAD SAFE]

You can use a base::WeakPtr and base::WeakPtrFactory (in base/memory/weak_ptr.h) to ensure that any invokes can not outlive the object they are being invoked on, without using reference counting. The base::Bind mechanism has special understanding for base::WeakPtr that will disable the task's execution if the base::WeakPtr has been invalidated. The base::WeakPtrFactory object can be used to generate base::WeakPtr instances that know about the factory object. When the factory is destroyed, all the base::WeakPtr will have their internal "invalidated" flag set, which will make any tasks bound to them to not dispatch. By putting the factory as a member of the object being dispatched to, you can get automatic cancellation.

 NOTE:  This only works when the task is posted to the same thread. Currently there is not a general solution that works for tasks posted to other threads. See the next section about CancelableTaskTracker for an alternative solution.
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

While base::WeakPtr is very helpful to cancel a task, it is not thread safe so can not be used to cancel tasks running on another thread. This is sometimes a performance critical requirement. E.g. We need to cancel database lookup task on DB thread when user changes inputed text. In this kind of situation CancelableTaskTracker is appropriate.

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