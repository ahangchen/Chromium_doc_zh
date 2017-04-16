# Threading

## Overview

Chromium is a very multithreaded product. We try to keep the UI as responsive as possible, and this means not blocking the UI thread with any blocking I/O or other expensive operations. Our approach is to use message passing as the way of communicating between threads. We discourage locking and threadsafe objects. Instead, objects live on only one thread, we pass messages between threads for communication, and we use callback interfaces (implemented by message passing) for most cross-thread requests.

The Thread object is defined in base/threading/thread.h. In general you should probably use one of the existing threads described below rather than make new ones. We already have a lot of threads that are difficult to keep track of. Each thread has a MessageLoop (see base/message_loop/message_loop.h) that processes messages for that thread. You can get the message loop for a thread using the Thread.message_loop() function.  More details about MessageLoop can be found in [Anatomy of Chromium MessageLoop](https://docs.google.com/document/d/1_pJUHO3f3VyRSQjEhKVvUU7NzCyuTCQshZvbWeQiCXU/edit#).

## Existing threads

Most threads are managed by the BrowserProcess object, which acts as the service manager for the main "browser" process. By default, everything happens on the UI thread. We have pushed certain classes of processing into these other threads. It has getters for the following threads:

* **ui_thread**: Main thread where the application starts up.
* **io_thread**: This thread is somewhat mis-named. It is the dispatcher thread that handles communication between the browser process and all the sub-processes. It is also where all resource requests (web page loads) are dispatched from (see [Multi-process Architecture](../Start_Here_Background_Reading/Multi-process_Architecture.md)).
* **file_thread**: A general process thread for file operations. When you want to do blocking filesystem operations (for example, requesting an icon for a file type, or writing downloaded files to disk), dispatch to this thread.
* **db_thread**: A thread for database operations. For example, the cookie service does sqlite operations on this thread. Note that the history database doesn't use this thread yet.
* **safe_browsing_thread**

Several components have their own threads:

* **History**: The history service object has its own thread. This might be merged with the db_thread above. However, we need to be sure that things happen in the correct order -- for example, that cookies are loaded before history since cookies are needed for the first load, and history initialization is long and will block it.
* **Proxy service**: See net/http/http_proxy_service.cc.
* **Automation proxy**: This thread is used to communicate with the UI test program driving the app.

## Keeping the browser responsive

As hinted in the overview, we avoid doing any blocking I/O on the UI thread to keep the UI responsive.  Less apparent is that we also need to avoid blocking I/O on the IO thread.  The reason is that if we block it for an expensive operation, say disk access, then IPC messages don't get processed.  The effect is that the user can't interact with a page.  Note that asynchronous/overlapped IO are fine.

Another thing to watch out for is to not block threads on one another.  Locks should only be used to swap in a shared data structure that can be accessed on multiple threads.  If one thread updates it based on expensive computation or through disk access, then that slow work should be done without holding on to the lock.  Only when the result is available should the lock be used to swap in the new data.  An example of this is in PluginList::LoadPlugins (src/content/common/plugin_list.cc). If you must use locks, [here](https://www.chromium.org/developers/lock-and-condition-variable) are some best practices and pitfalls to avoid.

In order to write non-blocking code, many APIs in Chromium are asynchronous. Usually this means that they either need to be executed on a particular thread and will return results via a custom delegate interface, or they take a base::Callback<> object that is called when the requested operation is completed.  Executing work on a specific thread is covered in the PostTask section below.

## Getting stuff to other threads

### base::Callback<>, Async APIs, and Currying

A base::Callback<> (see the [docs in callback.h](http://src.chromium.org/viewvc/chrome/trunk/src/base/callback.h?content-type=text%2Fplain)) is templated class with a Run() method.  It is a generalization of a function pointer and is created by a call to base::Bind.  Async APIs often will take a base::Callback<> as a means to asynchronously return the  results of an operation.  Here is an example of a hypothetical FileRead API.
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
In the example above, base::Bind takes the function pointer &DisplayString and turns it into a base::Callback<void(const std::string& result)>. The type of the generated base::Callback<> is inferred from the arguments.  Why not just pass the function pointer directly?  The reason is base::Bind allows the caller to adapt function interfaces and/or attach extra context via Currying (http://en.wikipedia.org/wiki/Currying).  For instance, if we had a utility function DisplayStringWithPrefix that took an extra argument with the prefix, we use base::Bind to adapt the interface as follows.
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
This can be used in lieu of creating an adapter functions a small classes that holds prefix as a member variable.  Notice also that the "MyPrefix: " argument is actually a const char*, while DisplayStringWithPrefix actually wants a const std::string&.  Like normal function dispatch, base::Bind, will coerce parameters types if possible.  See "How arguments are handled by base::Bind()" below for more details about argument storage, copying, and special handling of references.

### PostTask

The lowest level of dispatching to another thread is to use the MessageLoop.PostTask and MessageLoop.PostDelayedTask (see base/message_loop/message_loop.h). PostTask schedules a task to be run on a particular thread.  A task is defined as a base::Closure, which is a typedef for a base::Callback<void(void)>. PostDelayedTask schedules a task to be run after a delay on a particular thread. A task is represented by the base::Closure typedef, which contains a Run() function, and is created by calling base::Bind().  To process a task, the message loop eventually calls base::Closure's Run function, and then drops the reference to the task object. Both PostTask and PostDelayedTask take a tracked_objects::Location parameter, which is used for lightweight debugging purposes (counts and primitive profiling of pending and completed tasks can be monitored in a debug build via the url about:objects). Generally the macro value FROM_HERE is the appropriate value to use in this parameter.

Note that new tasks go on the message loop's queue, and any delay that is specified is subject to the operating system's timer resolutions. This means that under Windows, very small timeouts (under 10ms) will likely not be honored (and will be longer). Using a timeout of 0 in PostDelayedTask is equivalent to calling PostTask, and adds no delay beyond queuing delay. PostTask is also used to do something on the current thread "sometime after the current processing returns to the message loop." Such a continuation on the current thread can be used to assure that other time critical tasks are not starved on this thread.

The following is an example of a creating a task for a function and posting it to another thread (in this example, the file thread):
```c++
void WriteToFile(const std::string& filename, const std::string& data);
BrowserThread::PostTask(BrowserThread::FILE, FROM_HERE,
                        base::Bind(&WriteToFile, "foo.txt", "hello world!"));
```
You should always use BrowserThread to post tasks between threads.  Never cache MessageLoop pointers as it can cause bugs such as the pointers being deleted while you're still holding on to them.  More information can be found [here](https://www.chromium.org/developers/design-documents/threading/suble-threading-bugs-and-patterns-to-avoid-them).

### base::Bind() and class methods.

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
### How arguments are handled by base::Bind().

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
## Callback cancellation

There are 2 major reasons to cancel a task (in the form of a Callback):
* You want to do something later on your object, but at the time your callback runs, your object may have been destroyed.
* When input changes (e.g. user input), old tasks become unnecessary. For performance consideration, you should cancel them.
See following about different approaches for cancellation.
### Important notes about cancellation

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