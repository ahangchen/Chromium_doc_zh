#Profile架构

这篇文章描述了一个进行中的设计重构，始于2012年1月。

注意：2013年六月之后，这篇文章需要更新。相关的类被重命名(s/ProfileKeyed/BrowserContextKeyed/)以及移动到components/browser_context_keyed_service中。

Chromium有许多与**Profile**挂钩的特性，所谓Profile，即一些与当前用户以及跨越多个浏览器window的当前chrome会话。在Chromium刚起步的时候，profile只有一些动态的部分：cookie jar包，历史记录数据库，书签数据库，以及与用户首选项相关的一些东西。在Chromium工程三年的时间里，Profile变成了各个特性的连接点，派生出了一些东西像Profile::GetInstantPromoCounter()或者Profile::GetHostContentSettingsMap()。直到这个文章完成时，在Profile里已经有58个纯虚函数了。

Profile应当是一个最小引用，即一种不拥有实体的句柄对象。



##设计目标

- **我们必须能够分段地转移到新的架构中。**每次转移一个服务和特性。我们不能停止地球的转动，不能在一瞬间转换所有的东西。写下这些东西的时候，我们已经将19个服务移出Profile了。
- 我们应当只对调用端做小的修改，在调用端，Profile被用于在问题中获取服务。
- 我们必须修复Profile移除这个问题。当我们开始这项工作时，Profile外只有一小部分对象，处于拆分目的手动整理它们是可接受的。但现在我们有了75个组件，我们知道手动拆分整理是不对的，正如这里所写的。有着这么多组件的话，我们不能再依赖手动整理了。
- 我们必须允许加入编译新特性或者移除旧特性。现在我们有一些chromium的分支，它们不包含在Windows/Mac/Linux Google Chrome标准构建中所有的特性，我们应当允许给出这样一种方式，让这些分支能在不把#ifdef profile.h和profile_impl.h搞得一团糟的情况下，成功编译。这些分支也有他们需要提供的服务。（允许chromium分支添加它们自己的服务也触及我们不能在Profile移除过程中依赖手动整理的原因。）
- 延伸目标：将不同的特性隔离到它们自己的.a或.so文件里，进一步减小我们奇葩的编译链接时间。

##BrowserContextKeyedServiceFactory
> 浏览器上下文关键服务工厂

###旧的方式：Profile接口和ProfileImpl实现
在以前的设计里，服务通常用Profile里的一个访问器来获得：

```c++
class ProfileImpl {
  public:
    virtual FooService* GetFooService();
  private:
    scoped_ptr<FooService> foo_service_;
};
```

在之前的系统里，Profile是由大部分是纯虚访问器组成的结构。Normal（正常），Incognito（匿名）和Testing（测试）profile。

在这个世界里，Profile是所有活动的中心。profile有用它所有的服务，并向外界传递出去。Profile拆分遵循ProfileImpl中对服务排序的任何原则。另外的分支如果想要增加自己的服务或移除不需要的服务，而不修改Profile接口，都是不可能的。

###新的方式：BrowserContextKeyedServiceFactory
我们不再让Profile拥有某个service，而是设计了专用的单例FooServiceFactory，比如这样一个最小实现：

```c++
class FooServiceFactory : public BrowserContextKeyedServiceFactory {
 public:
  static FooService* GetForProfile(Profile* profile);

  static FooServiceFactory* GetInstance();

 private:
  friend struct DefaultSingletonTraits<FooServiceFactory>;

  FooServiceFactory();
  virtual ~FooServiceFactory();

  // BrowserContextKeyedServiceFactory:
  virtual BrowserContextKeyedService* BuildServiceInstanceFor(
    content::BrowserContext* context) const OVERRIDE;
};
```
我们有一个通用的BrowserContextKeyedServiceFactory，它用一个由你的BuildServiceInstanceFor()方法提供的对象，执行与profile相关的大部分工作。BrowserContextKeyedServiceFactory为你提供了一个重写接口，让你在响应Profile生命周期事件时，管理你的Service对象的生命周期，并在service依赖的service关闭前，关闭它本身。

一个绝对最小工厂会提供下面的方法：
- 一个static GetInstance()方法，单例指向你的工厂。
- 一个构造函数，关联这个BrowserContextKeyedServiceFactory和ProfileDependencyManager实例，并做DependsOn()声明。
- 一个GetForProfile()方法，包装BrowserContextKeyedServiceFactory，将返回结果转换为你需要的返回值。
- 一个BuildServiceInstanceFor()方法，框架会为每个|profile|调用一次这个方法，它必须返回你的服务的一个合适的实例。

另外，BrowserContextKeyedServiceFactory为你的控制行为提供了这些另外的辅助：

- RegisterUserPrefs()：每个Profile在初始化和用户首选项注册的地方会调用它一次
- 默认情况下，BCKSF在给定一个Incognito profile时会返回NULL
  - 如果你重写ServiceRedirectedInIncognito()方法并返回true，它会返回与normal Profile相关的服务。
  - 如果你重写ServiceHasOwnInstanceInIncognito()并返回true，它会为incognito profile创建一个新的服务。
- 默认情况下，BCKSF会延迟创建你的service，如果你重写ServiceIsCreatedWithProfile()并返回true，你的service会与profile一同创建。

- BCKSF为你在单元测试时提供了多种方式来控制行为。查看头文件了解更多。
- BCKSF为你一种方式提供一种方式增加并固定移除的和释放的行为。


###A Few Types of Factories

Not all objects have the same lifecycle and memory management. The previous paragraph was a major simplification; there is a base class BrowserContextKeyedBaseFactory that defines the most general dependency stuff while BrowserContextKeyedServiceFactory is a specialization that deals with normal objects. There is a second RefcountedBrowserContextKeyedServiceFactory that gives slightly different semantics and storage for RefCountedThreadSafe objects.
###A Brief Interlude About Complexity

So the above, from an implementation standpoint is significantly more complex than what came before it. Is all this really worth it?

*Yes.*

We absolutely have to address the interdependency of services. As it stands today, we do not shut down profiles after they are no longer needed in multiprofile mode because our crash rate when shutting down a profile is too high to ship to users. We have about 75 components that plug into the profile lifecycle and their dependency graph is complex enough that our naive manual ordering can not handle the complexity. All of the overrideable behavior above exists because it was implemented per service, ad hoc and copy pasted.

We likewise need to make it easy for other chromium variants to add their own features/compile features out of their build.
###Dependency Management Overview

With that in mind, let's look at how dependency management works. There is a single ProfileDependencyManager singleton, which is what is alerted to Profile creation and destruction. A PKSF will register and unregister itself with the ProfileDependencyManager. The job of the ProfileDependencyManager is to make sure that individual services are created and destroyed in a safe ordering.

Consider the case of these three service factories:
```c++
AlphaServiceFactory::AlphaServiceFactory()
    : BrowserContextKeyedServiceFactory(ProfileDependencyManager::GetInstance()) {
}

BetaServiceFactory::BetaServiceFactory()
    : BrowserContextKeyedServiceFactory(ProfileDependencyManager::GetInstance()) {
  DependsOn(AlphaServiceFactory::GetInstance());
     }

GammaServiceFactory::GammaServiceFactory()
    : BrowserContextKeyedServiceFactory(ProfileDependencyManager::GetInstance()) {
  DependsOn(BetaServiceFactory::GetInstance());
     }
```
The explicitly stated dependencies in this simplified graph mean that the only valid creation order for services is [Alpha, Beta, Gamma] and the destruction order is [Gamma, Beta, Alpha]. The above is all you, as a user of the framework, have to do to specify dependencies.

Behind the scenes, ProfileDependencyManager takes the stated dependency edges, performs a Kahn topological sort, and uses that in CreateProfileServices() and DestroyProfileServices().
##The Five Minute Tutorial of How to Convert Your Code

1. ***Make Your Existing FooService derive from BrowserContextKeyedService.**
2. **If possible, make your FooService no longer refcounted.** Most of the refcounted objects that hang off of Profile appear to be that way because they aren't using [base::bind/WeakPtrFactory](Threading.md) instead of needing to own data on multiple threads. (In case you have a real reason for being a RefCountedThreadSafe, such as being accessed on multiple threads, derive your factory from RefcountedBrowserContextKeyedServiceFactory and everything should just work.)
3. **Build a simple FooServiceFactory derived from BrowserContextKeyedServiceFactory. **Your FooServiceFactory will be the main access point consumers will ask for FooService. BrowserContextKeyedServiceFactory gives you a bunch of virtual methods that control behavior.
  1. BrowserContextKeyedService* BrowserContextKeyedServiceFactory::BuildServiceInstanceFor(content::BrowserContext* context) is the only required method. Given a BrowserContext handle, return a valid FooService.
  2. You can control the incognito behavior with ServiceRedirectedInIncognito() and ServiceHasOwnInstanceInIncognito().
4. **Add your service to the** EnsureBrowserContextKeyedServiceFactoriesBuilt() **list in** chrome_browser_main_extra_parts_profiles.cc.
5. **Understand shutdown behavior. **For historical reasons, we have a two phase deletion process:
  1. Every BrowserContextKeyedService will first have its Shutdown() method called. Use this method to drop weak references to the Profile or other service objects.
  2. Every BrowserContextKeyedService is deleted and its destructor is run. Minimal work should be done here. Attempts to call any *ServiceFactory::GetForProfile() will cause an assertion in debug mode.
6. Change each instance of "profile_->GetFooService()" to "FooServiceFactory::GetForProfile(profile_)".
If you need an example of what the above looks like, try looking at these patches:
- [r100516](http://src.chromium.org/viewvc/chrome?view=rev&revision=100516): A simple example, adding a new ProfileKeyedService. This shows off a minimal ServiceFactory subclass.
- [r104806](http://src.chromium.org/viewvc/chrome?view=rev&revision=104806): plugin_prefs_factory.h gives an example of how to deal with things that are (and have to stay) refcounted. This patch also shows off how to move your preferences into your ProfileKeyedServiceFactory.

##Debugging Tips

###Using the dependency visualizer

Chrome has a built in method to dump the profile dependency graph to a file in [GraphViz](http://www.graphviz.org/) format. When you run chrome with the command line flag  --dump-browser-context-graph, chrome will write the dependency information to your /path/to/profile/browser-context-dependencies.dot file. You can then convert this text file with dot, which is part of GraphViz:
```
dot -Tpng /path/to/profile/browser-context-dependencies.dot > png-file.png
```
This will give you a visual graph like this (generated January 23rd, 2012, click through for full size):

![](graph5.png)

Graph as of Aug 15, 2012

###Crashes at Shutdown

If you get a stack that looks like this:
```
ProfileDependencyManager::AssertProfileWasntDestroyed()
ProfileKeyedServiceFactory::GetServiceForProfile()
MyServiceFactory::GetForProfile()
... [Probably a bunch of frames] ...
OtherService::~OtherService()
ProfileKeyedServiceFactory::ProfileDestroyed()
ProfileDependencyManager::DestroyProfileServices()
ProfileImpl::~ProfileImpl()
```
The problem is that OtherService is improperly depending on MyService. The framework asserts if you try to use a Shutdown()ed component.