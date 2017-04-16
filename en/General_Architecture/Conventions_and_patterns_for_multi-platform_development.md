# Conventions and patterns for multi-platform development

Chromium is a large and complex cross-platform product. We try to share as much code as possible between platforms, while implementing the UI and OS integration in the most appropriate way for each. While this gives a better user experience, it adds extra complexity to the code. This document describes the recommended practices for keeping such cross-platform code clean.

We use a variety of different file naming suffixes to indicate when a file should be used:
 
- Mac files use the _mac suffix for lower-level files and Cocoa (Mac UI) files use the _cocoa suffix.
- iOS files use the _ios suffix (although iOS also uses some specific _mac files).
- Linux files use _linux suffix for lower-level files, _gtk for GTK-specific files, and _x for X Windows (with no GTK) specific files.
- Windows files use the _win suffix.
- Posix files shared between Mac, iOS, and Linux use the _posix suffix.
- Files for Chrome's "Views" UI (on Windows and experimental GTK) layout system use the _views suffix.

The separate front-ends of the browser are contained in their own directories:

- Mac Cocoa: chrome/browser/ui/cocoa
- Linux GTK: chrome/browser/ui/gtk
- Windows Views (and the experimental GTK-views): chrome/browser/ui/views

The [Coding Style](https://www.chromium.org/developers/coding-style) page lists some stylistic rules affecting platform-specific defines.

## How to separate platform-specific code

### Small platform differences: #ifdefs

When you have a class with many shared functions or data members, but a few differences, use #ifdefs around the platform-specific parts. If there are no significant differences, it's easier for everybody to keep everything in one place.
### Small platform differences in the header, larger ones in the implementation: split the implementation

There may be cases where there are few header file differences, but significant implementation differences for parts of the implementation. For example, base/waitable_event.h defines a common API with a couple of platform differences.

With significant implementation differences, the implementation files can be split. The prevents you from having to do a lot of #ifdefs for the includes necessary for each platform and also makes it easier to follow (three versions each of a set of functions in a file can get confusing). There can be different .cc files for each platform, as in base/waitable_event_posix.cc that implements posix-specific functions. If there were cross-platform functions in this class, they would be put in a file called base/waitable_event.cc.

### Complete platform implementations and callers: separate implementations

When virtually none of the implementation is shared, implement the class separately for each platform in separate files.

If all implementations are in a cross-platform directory such as base, they should be named with the platform name, such as FooBarWin in base/foo_bar_win.h. This case will generally be rare since files in these cross-platform files are normally designed to be used by cross-platform code, and separate header files makes this impossible. In some places we've defined a commonly named class in different files, so PlatformDevice is defined in skia/ext/platform_device_win.h, skia/ext/platform_device_linux.h, and skia/ext/platform_device_mac.h. This is OK if you really need to refer to this class in cross-platform code. But generally, cases like this will fall into the following rule.

If the implementations live in platform-specific directories such as chrome/browser/ui/cocoa or chrome/browser/ui/views, there is no chance that the class will be used by cross-platform code. In this case, the classes and filenames should omit the platform name since it would be redundant. So you would have FooBar implemented in chrome/browser/ui/cocoa/foo_bar.h.

Don't create different classes with different names for each platform and typedef it to a shared name. We used to have this for PlatformCanvas, where it was a typedef of PlatformCanvasMac, PlatformCanvasLinux, or PlatformCanvasWin depending on the platform. This makes it impossible to forward-declare the class, which is an important tool for reducing dependencies.

### When to use virtual interfaces

In general, virtual interfaces and factories should not be used for the sole purpose of separating platform differences. Instead, it should be be used to separate interfaces from implementations to make the code better designed. This comes up mostly when implementing the view as separate from the model, as in TabContentsView or RenderWidgetHostView. In these cases, it's desirable for the model not to depend on implementation details of the view. In many cases, there will only be one implementation of the view for each platform, but gives cleaner separation and more flexibility in the future.

In some places like TabContentsView, the virtual interface has non-virtual functions that do things shared between platforms. Avoid this. If the code is always the same regardless of the view, it probably shouldn't be in the view in the first place.
### Implementing platform-specific UI

In general, construct platform specific user interface elements from other platform-specific user interface elements. For instance, the views-specific class BrowserView is responsible for constructing many of the browser dialog boxes. The alternative is to wrap the UI element in a platform-independent interface and construct it from a model via a factory. This is significantly less desirable as it confuses ownership: in most cases of construction by factory, the UI element returned ends up being owned by the model that created it. However in many cases the UI element is most easily managed by the UI framework to which it belongs. For example, a views::View is owned by its view hierarchy and is automatically deleted when its containing window is destroyed. If you have a dialog box views::View that implements a platform independent interface that is then owned by another object, the views::View instance now needs to explicitly tell its view hierarchy not to try and manage its lifetime.

e.g. prefer this:
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

// FooDialogView and FooDialogController are automatically cleaned up when the window is closed.
class FooDialogView : public views::View {
  ...
 private:
  scoped_ptr<FooDialogController> controller_; // Cross-platform state/control logic
  ...
}
```
to this:
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

  // Why bother keeping FooDialog or even FooDialogController around?
  // Most dialogs are very seldom used.
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

Sometimes this latter pattern is necessary, but these occasions are rare, and very well understood by the frontend team. When porting, consider converting cases of the latter model to the former model if the UI element is something simple like a dialog box.
