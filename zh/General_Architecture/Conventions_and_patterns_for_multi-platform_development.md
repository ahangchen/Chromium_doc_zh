# 跨平台开发的约定与模式

Chromium是一个巨大而复杂的跨平台产品。我们试图在不同平台间共享尽可能多的代码，同时为每个平台用最合适的方式实现UI和操作系统集成。这提供了一个更好的用户体验，但它给代码增加了额外的复杂度。这个文档描述了保持这种跨平台代码简洁性的推荐实践。

我们使用大量不同带后缀的文件来表示一个文件应该被使用的时机：

- Mac文件中，低层级文件使用_mac后缀，Cocoa（Mac UI）文件使用_cocoa后缀。
- iOS文件使用_ios后缀（尽快iOS使用一些特定的_mac文件）。
- Linux文件中，低层级文件使用_linux后缀，GTK相关文件使用_gtk后缀，X Windows（不使用GTK）特定文件使用_x后缀。
- Windows文件使用_win后缀。
- Mac，iOS和Linux共享的Posix文件使用_posix后缀。
- Chrome view UI相关布局系统文件（在Windows和实验室环境GTK上）使用_views后缀。

独立的浏览器后端文件放在他们自己的目录里：

- Mac Cocoa: chrome/browser/ui/cocoa
- Linux GTK: chrome/browser/ui/gtk
- Windows Views (和实验室GTK-views): chrome/browser/ui/views

[编码风格](https://www.chromium.org/developers/coding-style) 页面列出一些风格上影响平台相关定义的规则。

## 如何隔离平台相关代码

### 小的平台差异： #ifdefs

当你有一个有着许多共享函数或数据成员和些许不同之处的类，在平台相关的部分使用#ifdefs。如果没有显著的差异，这会让每个人将每件事隔离开更加容易。

### 小的平台差异在头文件处理，大的差异在实现中处理：片段实现

可能有这样的情况，头文件几乎没有差别，部分实现有巨大的实现差异。例如base/waitable_event.h定义了一个通用的有着大量平台差异的API。

有着显著的实现差异，实现文件可以被隔离出来。这可以避免你陷入一个必须在include必要文件中为每个平台写一大堆#ifdef，并且使得追踪源码更容易（三个版本的函数集的代码放在同一个文件里可能令人困惑）。每个平台可以有不同的.cc文件，正如base/waitable_event_posix.cc中实现posix相关函数。如果在这个类里有跨平台的函数，他们应该被丢到一个名为base/waitable_event.cc的函数。

### 全平台实现和调用器：隔离实现

当抽象层面没有东西实现，就要在每个单独的文件里分别实现类。

如果所有的实现都在跨平台目录中，比如base，他们应该用平台的名字命名，比如base/foo_bar_win.h中的FooBarWin。这种例子通常很少，因为这些跨平台的文件通常设计用于跨平台代码，独立的头文件使得这种例子变得不可能。在一些地方，我们已经在不同的文件里定义了一个普通命名的类，所以PlatformDevice定义在skia/ext/platform_device_win.h, skia/ext/platform_device_linux.h, and skia/ext/platform_device_mac.h。如果你真的需要在跨平台代码里引用这个类，这是OK的。但通常，这种例子会变得遵循下面的这种规则。

如果实现存在于平台相关目录，比如chrome/browser/ui/cocoa或chrome/browser/ui/views，这个类就没有机会用于跨平台代码了。这种情况下，这个类和文件名应该忽略平台的名字，因为它是多余的。所以FooBar是在chrome/browser/ui/cocoa/foo_bar.h中实现的。

不要为每个平台创建不同的类，又把它们用typedef定义为同一个名字。我们曾经在PlatformCanvas上使用这种套路，根据平台，它被typedef为PlatformCanvasMac, PlatformCanvasLinux, 或 PlatformCanvasWin。这样就不可能提前声明这个类，而这是一个减小依赖的重要工具。

### 什么时候使用抽象的接口
通常，抽象接口与工厂不应该作为隔离平台差异的唯一目的。相反的，它应该用于将接口与优化代码设计的实现隔离开来。这最经常出现在从model中抽离view的实现中，比如TabContentsView或者RenderWidgetHostView。在这些例子里，model不依赖view的实现是有必要的。在许多情况下，多个平台的view只有一个实现，但为将来的开发提供了更干净的隔离与更多的可扩展性。

在有些地方，像TabContentsView，抽象层没有非抽象的、在平台间共享的函数。避免这种写法。如果不同view之间的代码总是一样，它可能首先就不应该在view中。

### 实现平台相关的UI

通常，从已有的平台相关的用户界面元素构建其他平台相关的用户界面元素。例如，view相关的类BrowserView负责构建许多浏览器对话框盒子。一种方法是，在一个平台无关的接口里包装UI元素，然后通过一个工厂，从一个model构造出它来。这是相当不必要的，因为它让迷乱了归属关系：大多数工厂构造的例子里，UI元素最后归属于创建它的model。然而在许多例子里，UI元素最容易由它所属的UI框架管理。例如，一个views::View归属于它的view层级，并且在包含它的window被销毁时，会自动被销毁。如果有一个对话框 views::View实现了一个平台无关的接口，然后被另一个对象拥有，那么views::View实例现在需要显式地告诉它的view层级不要去干涉它的生命周期。

e.g. 推荐这种写法:
```c++
// browser.cc:

Browser::ExecuteCommand(..) {
  ...
  case IDC_COMMAND_EDIT_FOO:
    window()->ShowFooDialog();
    break;
  ...
}

// browser_window.h:

class BrowserWindow {
...
  virtual void ShowFooDialog() = 0;
...
};

// browser_view.cc:

BrowserView::ShowFooDialog() {
  views::Widget::CreateWindow(new FooDialogView)->Show();
}

// foo_dialog_view.cc:

// FooDialogView和FooDialogController在window被关闭的时候会被自动清理
class FooDialogView : public views::View {
  ...
 private:
  scoped_ptr<FooDialogController> controller_; // 跨平台状态控制逻辑
  ...
}
```
不推荐这种
```c++

// browser.cc:

Browser::ExecuteCommand(..) {
  ...
  case IDC_COMMAND_EDIT_FOO: {
    FooDialogController::instance()->ShowUI();
    break;
  }
  ...
}

// foo_dialog_controller.h:

class FooDialog {
 public:
  static FooDialog* CreateFooDialog(FooDialogController* controller);
  virtual void Show() = 0;
  virtual void Bar() = 0;
};

class FooDialogController {
 public:
  ...
  static FooDialogController* instance() {
    static FooDialogController* instance = NULL;
    if (!instance)
      instance = Singleton<FooDialogController>::get();
    return instance;
  }
  ...
 private:
  ...
  void ShowUI() {
    if (!dialog_.get())
      dialog_.reset(FooDialog::CreateFooDialog(this));
    dialog_->Show();
  }

  // 为什么要把FooDialog或者甚至FooDialogController放在外面?
  // 大多数dialog起始很少被用到
  scoped_ptr<FooDialog> dialog_;
};

// foo_dialog_win.cc:

class FooDialogView : public views::View,
                      public FooDialogController {
 public:
  ...
  explicit FooDialogView(FooDialogController* controller) {
    set_parent_owned(false); // Now necessary due to scoped_ptr in FooDialogController.
  }
  ...
};

FooDialog* FooDialog::CreateFooDialog(FooDialogController* controller) {
  return new FooDialogView(controller);
}
```
有时候后一种模式是必要的，但这些情况很稀少，并且非常容易被前端的团队所理解。移植的时候，如果UI元素有时候像dialog box一样简单的话，考虑把后一种模式转为前一种。

