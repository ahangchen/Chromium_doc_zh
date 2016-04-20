 ###[Chromium整体架构](README.md)
 将Chromium设计文档页面的内容整理成markdown格式，并翻译成中文，整体架构方面已翻译完成。
 
 原本是出于了解更多关于Android Webview的相关知识的目的开始翻译，但学到的更多的是架构设计方面的内容。
 
 > 翻译了这个部分后发现工作量挺大的，希望感兴趣的同学[一起来翻译](https://github.com/ahangchen/Chromium_doc_zh)，共同进步。
 
  - [跨平台开发的约定与模式](Conventions_and_patterns_for_multi-platform_development.md)
  - [扩展安全架构](Extension_Security_Architecture.md): 扩展系统是如何降低扩展脆弱性的严重程度的
  - [硬件视频加速](HW_Video_Acceleration_in_Chrom{eium}{OS}.md)
  - [跨进程通信](Inter-process_Communication.md): 浏览进程，绘制器，插件进程是如何交流的
  - [多进程资源加载](Multi-process_Resource_Loading.md): 页面和图像是如何从网络加载到绘制器中的
  - [插件架构](Plugin_Architecture.md)
  - [进程模型](Process_Models.md): Our 创建新绘制进程的策略
  - [Profile架构](Profile_Architecture.md)
  - [安全浏览](SafeBrowsing.md)
  - [沙箱](Sandbox.md)
    - [沙箱FAQ](Sandbox_FAQ.md)
    - [OSX中的沙箱设计](OSX_Sandbox_design.md)
  - [安全架构](Security_Architecture.md): Chromium沙箱绘制引擎是如何保护免受恶意软件侵害的
  - [启动](Startup.md)
  - [线程](Threading.md): 在chromium中如何使用线程
  
 也可以看看 [V8](http://code.google.com/apis/v8/)的文档, 这是Chromium使用的Javascript引擎
