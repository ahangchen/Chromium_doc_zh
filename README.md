# Chromium_doc_zh
> Chromium中文文档 for https://www.chromium.org/developers/design-documents

> 翻译之加强对android webview理解，同时作为架构设计的学习。

> 还未完全完成，不断更新ing，欢迎star

> gitbook地址：https://ahangchen.gitbooks.io/chromium_doc_zh/content/zh/ 适合阅读，可以导出pdf

> github地址： https://github.com/ahangchen/Chromium_doc_zh 

> 翻译了一段时间感觉内容真的好多，欢迎志同道合的朋友一起翻译。

# 设计文档

- [Start Here: 背景阅读](zh/Start_Here_Background_Reading/README.md): 描述Chromium的宏观架构
  - [多进程架构](zh/Start_Here_Background_Reading/Multi-process_Architecture.md)
  
    **Note**: 设计文档的大部分剩余部分都认为你对上面这个文档里的内容非常熟悉。

  - [Chromium如何展示web界面](zh/Start_Here_Background_Reading/How_Chromium_Displays_Web_Pages.md): 自底向上概述WebKit是如何嵌入到Chromium中的
  
## See Also: 源代码中的设计文档

    https://chromium.googlesource.com/chromium/src/+/master/docs/



- ###[整体架构](zh/General_Architecture/README.md)
  - [跨平台开发的约定与模式](zh/General_Architecture/Conventions_and_patterns_for_multi-platform_development.md)
  - [扩展安全架构](zh/General_Architecture/Extension_Security_Architecture.md): 扩展系统是如何降低扩展脆弱性的严重程度的
  - [硬件视频加速](zh/General_Architecture/HW_Video_Acceleration_in_Chrom{eium}{OS}.md)
  - [跨进程通信](zh/General_Architecture/Inter-process_Communication.md): 浏览进程，绘制器，插件进程是如何交流的
  - [多进程资源加载](zh/General_Architecture/Multi-process_Resource_Loading.md): 页面和图像是如何从网络加载到绘制器中的
  - [插件架构](zh/General_Architecture/Plugin_Architecture.md)
  - [进程模型](zh/General_Architecture/Process_Models.md): 创建新绘制进程的策略
  - [Profile架构](zh/General_Architecture/Profile_Architecture.md)
  - [安全浏览](zh/General_Architecture/SafeBrowsing.md)
  - [沙箱](zh/General_Architecture/Sandbox.md)
    - [沙箱FAQ](zh/General_Architecture/Sandbox_FAQ.md)
    - [OSX中的沙箱设计](zh/General_Architecture/OSX_Sandbox_design.md)
  - [安全架构](zh/General_Architecture/Security_Architecture.md): Chromium沙箱绘制引擎是如何保护免受恶意软件侵害的
  - [启动](zh/General_Architecture/Startup.md)
  - [线程](zh/General_Architecture/Threading.md): 在chromium中如何使用线程
  
 也可以看看 [V8](zh/http://code.google.com/apis/v8/)的文档, 这是Chromium使用的Javascript引擎

- ### [UI Framework](zh/UI_Framework/README.md)
  - [UI开发实践](zh/UI_Framework/UI_Development_Practices.md): 在Chrome的content区域内外开发的最佳实践
  - [Views framework](zh/UI_Framework/Views_framework.md): Windows和Chrome OS上使用的UI layout 层级
  - [views Windowing系统](zh/UI_Framework/views_Windowing_system.md): 如何用view构建对话框盒子和其他windowUI
  - [Aura](zh/UI_Framework/Aura.md): Chrome下一代硬件加速UI框架，新的ChromeOS 系统由它构建而成
  - [Native控制](zh/UI_Framework/NativeControls.md): 在view中使用平台原生widget
  - 用View和Aura实现聚焦与激活
- ### [Graphics](zh/Graphics/README.md)
  - [概述](zh/Graphics/Overview.md)
  - [Chrome中使用的GPU加速](zh/Graphics/GPU_Accelerated_Compositing_in_Chrome.md)
  - [GPU特性状态仪表盘](zh/Graphics/GPU_Feature_Status_Dashboard.md)
  - [绘制架构图](zh/Graphics/Rendering_Architecture_Diagrams.md)
  - [Graphics和Skia](zh/Graphics/Graphics_and_Skia.md)
  - [绘制文本以及Chrome UI 文本绘制](zh/Graphics/RenderText_and_Chrome_UI_text_drawing.md)
  - [GPU命令缓冲区](zh/Graphics/GPU_Command_Buffer.md)
  - [GPU程序缓冲区](zh/Graphics/GPU_Program_Caching.md)
  - [Blink/WebCore中的组合](zh/Graphics/Compositing_in_Blink_WebCore.md)
  - [排版线程架构](zh/Graphics/Compositor_Thread_Architecture.md)
  - [绘制标准](zh/Graphics/Rendering_Benchmarks.md)
  - [实现层绘制](zh/Graphics/Impl-side_Painting.md)
  - [视频回放与排版](zh/Graphics/Video_Playback_and_Compositor.md)
  - [ANGLE架构表示](zh/Graphics/ANGLE_architecture_presentation.md)
- [网络栈](zh/Network_stack/README.md)
  - [概述](zh/Network_stack/Overview.md)
  - [网络栈的目标](zh/Network_stack/Network_Stack_Objectives.md)
  - [Crypto](zh/Network_stack/Crypto.md)
  - [磁盘缓存](zh/Network_stack/Disk_Cache.md)
  - [HTTP缓存](zh/Network_stack/HTTP_Cache.md)
  - [进程外代理解决草案[unimplemented]](zh/Network_stack/Out_of_Process_Proxy_Resolving_Draft_[unimplemented].md)
  - [代理设置与回退](zh/Network_stack/Proxy_Settings_and_Fallback.md)
  - [调试网络代理问题](zh/Network_stack/Debugging_network_proxy_problems.md)
  - [HTTP授权](zh/Network_stack/HTTP_Authentication.md)
  - [查看网络内部情况的工具](zh/Network_stack/View_network_internals_tool.md)
  - [用SPDY页面使得网络更快](zh/Network_stack/Make_the_web_faster_with_SPDY_pages.md)
  - [用OUIC页面使得网络更快](zh/Network_stack/_the_web_even_faster_with_QUIC_pages.md)
  - [Cookie存储与获取](zh/Network_stack/Cookie_storage_and_retrieval.md)
- [安全](zh/Security/README.md)
  - [概述](zh/Security/Security_Overview.md)
  - [保护缓存用户数据](zh/Security/Protecting_Cached_User_Data.md)
  - [系统强化](zh/Security/System_Hardening.md)
  - [Chaps技术设计](zh/Security/Chaps_Technical_Design.md)
  - [TPM使用](zh/Security/TPM_Usage.md)
  - [每个页面的子源](zh/Security/Per-page_Suborigins.md)
  - [加密分割恢复](zh/Security/Encrypted_Partition_Recovery.md)
- [Input](zh/Input/README.md)
  - 看这个文档[chromium input](zh/Input/chromium_input.md)（关于设计文档以及一些其他资源）
- [绘制](zh/Rendering/README.md)
  - [多列布局](zh/Rendering/Multi-column_layout.md)
  - [刷新时重置Style](zh/Rendering/Style_Invalidation_in_Blink.md)
  - [刷新与空间的协调](zh/Rendering/Blink_Coordinate_Spaces.md)
- [构建](zh/Building/README.md)
  - [IDL构建](zh/Building/IDL_build.md)
  - [IDL编译器](zh/Building/IDL_compiler.md)
  - 也可以看这个文档，[GYP, the build script generation tool.](zh/Building/GYP_the_build_script_generation_tool..md)
- [测试](zh/Testing/README.md)
  - [Layout测试结果面板](zh/Testing/Layout_test_results_dashboard.md)
  - [Test shell中的真实主题](zh/Testing/Generic_theme_for_Test_Shell.md)
  - [完全回退移除layout测试](zh/Testing/Moving_LayoutTests_fully_upstream.md)
- [特性相关](zh/Feature-Specific/README.md)
  - [关于冲突](zh/Feature-Specific/aboutconflicts.md)
  - [可用性](zh/Feature-Specific/Accessibility.md): 当前（以及将来）可用性支持的轮廓。
  - [自适应屏幕截图与镜像](zh/Feature-Specific/Auto-Throttled_Screen_Capture_and_Mirroring.md)
  - [浏览器window](zh/Feature-Specific/Browser_Window.md)
  - [Chromium打印代理](zh/Feature-Specific/Chromium_Print_Proxy.md): 为保留打印机与未来的云服务打印机使能云打印服务
  - [强制弹出窗口](zh/Feature-Specific/Constrained_Popup_Windows.md)
  - [桌面通知](zh/Feature-Specific/Desktop_Notifications.md)
  - [Chrome on Windows的直写式Cache](zh/Feature-Specific/DirectWrite_Font_Cache_for_Chrome_on_Windows.md)
  - [DNS预拉取](zh/Feature-Specific/DNS_Prefetching.md): 通过在用户打开链接前,预先解析域名,来减少延迟
  - [在浏览器窗口中,嵌入式Flash的全屏](zh/Feature-Specific/Embedding_Flash_Fullscreen_in_the_Browser_Window.md)
  - [拓展: 设计文档与推荐的api ](zh/Feature-Specific/Extensions_Design_documents_and_proposed_APIs..md): Design documents and proposed APIs.
  - [查找栏](zh/Feature-Specific/Find_Bar.md)
  - [表单自动填充](zh/Feature-Specific/Form_Autofill.md): 一个用合适的数据自动填充html表单的特性
  - [地理信息](zh/Feature-Specific/Geolocation.md): 添加对[W3C Geolocation API](zh/http://www.w3.org/TR/geolocation-API/)的支持,使用native WebKit bindings.
  - [IDN in Google Chrome](zh/Feature-Specific/IDN_in_Google_Chrome.md)
  - [索引式DB(早期草稿)](zh/Feature-Specific/IndexedDB__early_draft_.md)
  - [信息栏](zh/Feature-Specific/Info_Bars.md)
  - [安装](zh/Feature-Specific/Installer.md): 注册入口与快捷方式
  - [即时](zh/Feature-Specific/Instant.md)
  - [独立网站](zh/Feature-Specific/Isolated_Sites.md)
  - [Linux资源与本地化字符串](zh/Feature-Specific/Linux_Resources_and_Localized_Strings.md): Linux资源与本地化字符串的加载
  - [媒体路由 & Web Presentation API](zh/Feature-Specific/Media_Router_&_Web_Presentation_API.md)
  - [内存使用统计后端](zh/Feature-Specific/Memory_Usage_Backgrounder.md): 我们在Chromium中如何测量内存的一些api
  - [鼠标锁定](zh/Feature-Specific/Mouse_Lock.md)
  - [地址栏自动完成](zh/Feature-Specific/Omnibox_Autocomplete/README.md): 在地址栏中打字时,Chromium搜索并建议可能的结果
    - [快速提供历史](zh/Feature-Specific/Omnibox_Autocomplete/HistoryQuickProvider.md): 由用户历史访问网站提供建议
  - [搜索栏/IME协作](zh/Feature-Specific/Omnibox_IME_Coordination.md)
  - [Ozone移植抽象](zh/Feature-Specific/Ozone_Porting_Abstraction.md)
  - [密码生成](zh/Feature-Specific/Password_Generation.md)
  - [Pepper插件实现](zh/Feature-Specific/Pepper_plugin_implementation.md)
  - [插件能力保存](zh/Feature-Specific/Plugin_Power_Saver.md)
  - [选项](zh/Feature-Specific/Preferences.md)
  - [预渲染](zh/Feature-Specific/Prerender.md)
  - [打印预览](zh/Feature-Specific/Print_Preview.md)
  - [打印](zh/Feature-Specific/Printing.md)
  - [view中基于矩形的事件目标](zh/Feature-Specific/Rect-based_event_targeting_in_views.md): 使得触摸激发view元素更加容易
  - [替换语义cookie提示](zh/Feature-Specific/Replace_the_modal_cookie_prompt.md)
  - [安全搜索](zh/Feature-Specific/SafeSearch.md)
  - [Sane Time](zh/Feature-Specific/Sane_Time.md): 在Chrome中决定一个精确的时间
  - [安全web代理](zh/Feature-Specific/Secure_Web_Proxy.md)
  - [服务进程](zh/Feature-Specific/Service_Processes.md)
  - [站点隔离](zh/Feature-Specific/Site_Isolation.md): 进程内的一些工作,提高Chromium在网站安全方面的进程模型
  - [软件更新: Courgette](zh/Feature-Specific/Software_Updates_Courgette.md)
  - [同步](zh/Feature-Specific/Sync.md)
  - [Tab助手](zh/Feature-Specific/Tab_Helpers.md)
  - [Tab搜索](zh/Feature-Specific/Tab_to_search.md): 如果让地址栏自动提供标签来搜索你的网站
  - [Tabtastic2需求](zh/Feature-Specific/Tabtastic2_Requirements.md)
  - [临时下载](zh/Feature-Specific/Temporary_downloads.md)
  - [时间资源](zh/Feature-Specific/Time_Sources.md): 在一个Chrome OS设备上决定时间
  - [时钟](zh/Feature-Specific/TimeTicks.md): 我们单一的时钟是如何在不同系统上工作的
  - [UI镜像基础设施](zh/Feature-Specific/UI_Mirroring_Infrastructure.md): 描述ChromeViews中的UI框架,允许在希伯来与或阿拉伯语这样的RTL语言环境中镜像浏览器UI
  - [UI定位](zh/Feature-Specific/UI_Localization.md): 描述如何定位要加入chromium的字符串
  - [用户脚本](zh/Feature-Specific/User_scripts.md): Chrome对于用户脚本的一些支持信息
  - [视频](zh/Feature-Specific/Video.md)
  - [WebSocket](zh/Feature-Specific/WebSocket.md): 允许web应用程序与服务端进程维护一个双向的交流
  - [Web MIDI](zh/Feature-Specific/Web_MIDI.md)
  - [Web导航 API内部实现](zh/Feature-Specific/WebNavigation_API_internals.md)
- [OS-相关](zh/OS-Specific/README.md)
  - [Android](zh/OS-Specific/Android/README.md)
    - [Android上的Java资源](zh/OS-Specific/Android/Java_Resources_on_Android.md)
    - [JNI绑定](zh/OS-Specific/Android/JNI_Bindings.md)
    - [WebView代码组织](zh/OS-Specific/Android/WebView_code_organization.md)
  - [Chrome OS](zh/OS-Specific/Chrome_OS/README.md)
    - 查看[Chrome OS设计文档相关章节](zh/OS-Specific/Chrome_OS/Chrome_OS_design_documents_section..md)
  - [Mac OS X](zh/OS-Specific/Mac_OS_X/README.md)
    - [Apple脚本Support](zh/OS-Specific/Mac_OS_X/AppleScript_Support.md)
    - [BrowserWindowController对象所有权](zh/OS-Specific/Mac_OS_X/BrowserWindowController_Object_Ownership.md)
    - [确认退出](zh/OS-Specific/Mac_OS_X/Confirm_to_Quit.md)
    - [Mac App模式(草案)](zh/OS-Specific/Mac_OS_X/Mac_App_Mode__Draft_.md)
    - [Mac全屏模式(草案)](zh/OS-Specific/Mac_OS_X/Mac_Fullscreen_Mode__Draft_.md)
    - [Mac NPAPI插件托管](zh/OS-Specific/Mac_OS_X/Mac_NPAPI_Plugin_Hosting.md)
    - [Mac上UI定位的相关记录](zh/OS-Specific/Mac_OS_X/Mac_specific_notes_on_UI_Localization.md)
    - [菜单,快捷键和命令调度](zh/OS-Specific/Mac_OS_X/Menus_Hotkeys_&_Command_Dispatch.md)
    - [IOSurface使用与语义相关会议的一些记录](zh/OS-Specific/Mac_OS_X/Notes_from_meeting_on_IOSurface_usage_and_semantics.md)
    - [OS X跨进程通信(已过时)](zh/OS-Specific/Mac_OS_X/OS_X_Interprocess_Communication__Obsolete_.md)
    - [密码管理/密码链集成](zh/OS-Specific/Mac_OS_X/Password_Manager_Keychain_Integration.md)
    - [沙箱设计](zh/OS-Specific/Mac_OS_X/Sandboxing_Design.md)
    - [Tab切换设计(包括标签布局与标签拖动)](zh/OS-Specific/Mac_OS_X/Tab_Strip_Design__Includes_tab_layout_and_tab_dragging_.md)
    - [扳手状菜单按钮](zh/OS-Specific/Mac_OS_X/Wrench_Menu_Buttons.md)
- [Other](zh/Other/README.md)
  - [64位支持](zh/Other/64-bit_Support.md)
  - [浏览器组件/层级组件](zh/Other/Browser_Components___Layered_Components.md)
  - [闭合编译Chrome代码](zh/Other/Closure_Compiling_Chrome_Code.md)
  - [内容模块/内容API](zh/Other/content_module___content_API.md)
  - [正在编写的设计文档(wiki)](zh/Other/Design_docs_that_still_need_to_be_written__wiki_.md)
  - [可移植性:进程内重构关键浏览器进程架构](zh/Other/In_progress_refactoring_of_key_browser-process_architecture_for_porting.md)
  - [网络实验](zh/Other/Network_Experiments.md)
  - [将float的inlinebox转为layout单元](zh/Other/Transitioning_InlineBoxes_from_floats_to_LayoutUnits.md)

> 如果觉得我的文章对您有帮助的话，不妨点个star




