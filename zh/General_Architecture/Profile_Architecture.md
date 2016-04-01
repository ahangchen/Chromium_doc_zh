#Profile架构

这篇文章描述了一个进行中的设计重构，始于2012年1月。

注意：2013年六月之后，这篇文章需要更新。相关的类被重命名(s/ProfileKeyed/BrowserContextKeyed/)以及移动到components/browser_context_keyed_service中。

Chromium有许多与**Profile**挂钩的特性，所谓Profile，即一些与当前用户以及跨越多个浏览器window的当前chrome会话。当Chromium第一次打开的时候，profile只有一些动态的部分：cookie jar包，历史记录数据库，书签数据库，以及与用户首选项相关的一些东西。在Chromium工程三年的时间里，Profile变成了各个特性的连接点，派生出了一些东西像Profile::GetInstantPromoCounter()或者Profile::GetHostContentSettingsMap()。直到这个文章完成时，在Profile里已经有58个纯虚函数了。

Profile应当是一个最小引用，即一种不拥有实体的句柄对象。



##设计目标

- **我们必须能够分段地转移到新的架构中。**每次转移一个服务和特性。我们不能停止地球的转动，不能在一瞬间转换所有的东西。写下这些东西的时候，我们已经将19个服务移出Profile了。
- 我们应当只对调用端做小的修改，在调用端，Profile被用于在问题中获取服务。
- 我们必须修复Profile崩溃问题。当我们开始这项工作时，Profile外只有一小部分对象，处于拆分目的手动整理它们是可接受的。但现在我们有了75个组件，我们知道手动拆分整理是不对的，正如这里所写的。有着这么多组件的话，我们不能再依赖手动整理了。
- 我们必须允许加入编译新特性或者移除旧特性。现在我们有一些chromium的分支，它们不包含标准Windows/Mac/Linux Google Chrome构建中所有的特性。
- We must fix Profile shutdown. When we started and only had a few objects hanging off of Profile, manual ordering was acceptable for destruction. Now we have over seventy five components and we know that our manual destruction ordering is incorrect as written today. We can not rely on manual ordering when we have so many components.
- We must allow features to be compiled in and out. Now that we have chromium variants that don't contain all the features in a standard Windows/Mac/Linux Google Chrome build, we need a way to allow these variants to compile without #ifdefing profile.h and profile_impl.h into a mess. These variants also have their own services that they'd like to provide. (Letting chromium variants add their own services also touches on why we can't rely on manual ordering in Profile shutdown.)
  - Stretch goal: Separate features go in their own .a/.so files to further minimize our ridiculous linking time.

##BrowserContextKeyedServiceFactory

###The Old Way: Profile interface and ProfileImpl

In the previous design, services were fetched through an accessor on Profile:
```c++
class ProfileImpl {
  public:
    virtual FooService* GetFooService();
  private:
    scoped_ptr<FooService> foo_service_;
};
```

In the previous system, Profile was an interface with mostly pure virtual accessors. There were separate versions of Profile for Normal, Incognito and Testing profiles.

In this world, the Profile was the center of all activity. The profile owned all of its service and handed them out. Profile destruction was according to whatever order the services were listed in ProfileImpl. There wasn't a way for another variant to add its own services (or leave out ones it didn't need) without modifying the Profile interface.

###The New Way: BrowserContextKeyedServiceFactory

Instead of having the Profile own FooService, we have a dedicated singleton FooServiceFactory, like this minimal one:
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
We have a generalized BrowserContextKeyedServiceFactory which performs most of the work of associating a profile with an object provided by your BuildServiceInstanceFor() method. The BrowserContextKeyedServiceFactory provides an interface for you to override while managing the lifetime of your Service object in response to Profile lifetime events and making sure your service is shut down before services it depends on.

An absolutely minimal factory will supply the following methods:
- A static GetInstance() method that refers to your Factory as a Singleton.
- A constructor that associates this BrowserContextKeyedServiceFactory with the ProfileDependencyManager singleton, and makes DependsOn() declarations.
- A GetForProfile() method that wraps BrowserContextKeyedServiceFactory, casting the result back to whatever type you need to return.
- A BuildServiceInstanceFor() method which is called once by the framework for each |profile|, which must return a proper instance of your service.

In addition, BrowserContextKeyedServiceFactory provides these other knobs for how you can control behavior:
- RegisterUserPrefs() is called once per Profile during initialization and is where you can place any user pref registration.
- By default, BCKSF return NULL when given an Incognito profile.
  - If you override ServiceRedirectedInIncognito() to return true, it will return the associated normal Profile's service.
  - If you override ServiceHasOwnInstanceInIncognito() to return true, it will create a new service for the incognito profile.
- By default, BCKSF will lazily create your service. If you override ServiceIsCreatedWithProfile() to return true, your service will be created alongside the profile.
- BCKSF gives you multiple ways to control behavior during unit tests. See the header for more details.
- BCKSF gives you a way to augment and tweak the shutdown and deallocation behavior.

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