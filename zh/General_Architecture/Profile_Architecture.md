# Profile架构

这篇文章描述了一个进行中的设计重构，始于2012年1月。

注意：2013年六月之后，这篇文章需要更新。相关的类被重命名(s/ProfileKeyed/BrowserContextKeyed/)以及移动到components/browser_context_keyed_service中。

Chromium有许多与**Profile**挂钩的特性，所谓Profile，即一些与当前用户以及跨越多个浏览器window的当前chrome会话。在Chromium刚起步的时候，profile只有一些动态的部分：cookie jar包，历史记录数据库，书签数据库，以及与用户首选项相关的一些东西。在Chromium工程三年的时间里，Profile变成了各个特性的连接点，派生出了一些东西像Profile::GetInstantPromoCounter()或者Profile::GetHostContentSettingsMap()。直到这个文章完成时，在Profile里已经有58个纯虚函数了。

Profile应当是一个最小引用，即一种不拥有实体的句柄对象。



## 设计目标

- **我们必须能够分段地转移到新的架构中。**每次转移一个服务和特性。我们不能停止地球的转动，不能在一瞬间转换所有的东西。写下这些东西的时候，我们已经将19个服务移出Profile了。
- 我们应当只对调用端做小的修改，在调用端，Profile被用于在问题中获取服务。
- 我们必须修复Profile移除这个问题。当我们开始这项工作时，Profile外只有一小部分对象，处于拆分目的手动整理它们是可接受的。但现在我们有了75个组件，我们知道手动拆分整理是不对的，正如这里所写的。有着这么多组件的话，我们不能再依赖手动整理了。
- 我们必须允许加入编译新特性或者移除旧特性。现在我们有一些chromium的分支，它们不包含在Windows/Mac/Linux Google Chrome标准构建中所有的特性，我们应当允许给出这样一种方式，让这些分支能在不把#ifdef profile.h和profile_impl.h搞得一团糟的情况下，成功编译。这些分支也有他们需要提供的服务。（允许chromium分支添加它们自己的服务也触及我们不能在Profile移除过程中依赖手动整理的原因。）
- 延伸目标：将不同的特性隔离到它们自己的.a或.so文件里，进一步减小我们奇葩的编译链接时间。

## BrowserContextKeyedServiceFactory
> 浏览器上下文关键服务工厂

### 旧的方式：Profile接口和ProfileImpl实现
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

### 新的方式：BrowserContextKeyedServiceFactory
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


### 几种工厂

并非所有对象都有一样的生命周期和内存管理。前面的段落是一个主要的简化版本；基类BrowserContextKeyedBaseFactory定义了大多数常见依赖部分，BrowserContextKeyedServiceFactory是一个具体处理通常对象的工厂。另一个RefcountedBrowserContextKeyedServiceFactory在语义上以及对RefCountedThreadSafe对象的存储上有轻微的差异。

## 关于复杂度的一个小插曲

上面的这些，在实现上比之前的版本要复杂许多，这是否值得呢？

*Yes.*

我们绝对应该强调服务的独立性。正如它今天的样子，在多profile模式不再有必要之后，我们没有马上去掉profile，因为在去掉profile时，我们的crash率太高了，不能为用户所接受。我们有75个组件插在profile的生命周期当中，他们之间的依赖图如此复杂以至于我们简单的手动整理不能处理这种复杂度。上面所有可重写的行为之所以存在，是因为它由每个服务，特定的广告，以及复制粘贴实现。

我们同样需要让其他chromium分支能够方便地添加他们自己的特性，或者排除它们的构建以外的特性。

### 依赖管理概览

考虑这一点，让我们看一下依赖管理是如何工作的。我们有ProfileDependencyManager的一个单例，它与Profile创建与销毁相关联。一个PKSF由ProfileDependencyManager来注册以及注销。ProfileDependencyManager的工作是确保各个服务用一种安全的方式创建与销毁。


考虑下面这个有者三个服务工厂的例子：
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

在这个简化的代码结构中，显式声明的依赖意味着这些服务唯一有效的创建顺序是[Alpha, Beta, Gamma],唯一有效的销毁顺序是[Gamma, Beta, Alpha]。上面的这些是你，也就是这个框架的使用者，所必须指定的依赖。

在幕后，ProfileDependencyManager管理所声明的依赖的关系，展示了一个Kahn的拓扑排序，并在CreateProfileServices()和DestroyProfileServices()中得到应用。


## 五分钟了解如何转换你的代码

1. **让你已有的FooService继承BrowserContextKeyedService。**
2. **可能的话，不要再让你的FooService得到引用计数了。**大多数与Profile相关的被引用计数的对象似乎因为他们没有使用[base::bind/WeakPtrFactory](Threading.md)，而需要在多线程使用自己的数据。（在这个例子里，线程安全的引用计数是有必要的，比如，多线程访问时，让你的工厂继承自RefcountedBrowserContextKeyedServiceFactory，这样一切都能正常工作。）
3. **构建一个简单继承自BrowserContextKeyedServiceFactory的FooServiceFactory。**消费者请求FooService时，你的FooServiceFactory将会是主要的访问点。
  1. BrowserContextKeyedService\* BrowserContextKeyedServiceFactory::BuildServiceInstanceFor(content::BrowserContext\* context)是唯一需要的函数。传入一个BrowserContext句柄，返回一个有效的FooService。
  2. 你可以用ServiceRedirectedInIncognito() 和 ServiceHasOwnInstanceInIncognito()控制incognito行为。
4. **把你的服务添加到**chrome_browser_main_extra_parts_profiles.cc中中的EnsureBrowserContextKeyedServiceFactoriesBuilt()**列表**。
5. **理解Shutdown行为。**出于历史原因，我们必须做两个阶段的Shutdown操作：
  1. 每个BrowserContextKeyedService首先要调用它的Shutdown()方法。使用这个方法来移除对Profile或其他服务对象的弱引用。
  2. 删除每个BrowserContextKeyedService，运行它的析构器。最小化的工作需要在这里完成。调用任何\*ServiceFactory::GetForProfile()会在调试模式下触发的一个断言。
6. 将每个"profile_->GetFooService()"实例改为"FooServiceFactory::GetForProfile(profile_)"。

如果你需要上面这些步骤的例子，可以看看这些补丁：
- [r100516](http://src.chromium.org/viewvc/chrome?view=rev&revision=100516): 一个简单的例子，添加了一个新的ProfileKeyedService。这展示了一个最小的ServiceFactory子类。
- [r104806](http://src.chromium.org/viewvc/chrome?view=rev&revision=104806): plugin_prefs_factory.h给出了一个例子，阐述了如何处理（必须）引用计数的东西。 这个补丁也展示了如何将你的首选项移到你的ProfileKeyedServiceFactory中。

## 调试技巧

### 使用依赖抽象器

Chrome有一个内置的方法来导出profile依赖图，生成一个[GraphViz](http://www.graphviz.org/)格式的文件。当你命令行运行chrome，附带--dump-browser-context-graph标记时，chrome会将依赖信息写到你的/path/to/profile/browser-context-dependencies.dot文件。然后你可以用dot转化这个文件，dot是GraphViz的一个部分：
```
dot -Tpng /path/to/profile/browser-context-dependencies.dot > png-file.png
```
这会给你一个像下面这样的抽象图(2012年1月23日生成，点击查看大图):

![](graph5.png)



### Shutdown时的crash

如果出现了一个这样的栈：
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
问题就是，OtherService没有正确地依赖MyService。在你使用Shutdown()组件时，框架会触发一个assert。
