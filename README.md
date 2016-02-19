# Chromium_doc_zh
Chromium中文文档 for

https://www.chromium.org/developers/design-documents

翻译之加强对android webview理解

#Design Documents

- [Start Here: Background Reading](Start_Here_Background_Reading/README.md): Describes the high-level architecture of Chromium 
  - [Multi-process Architecture](Start_Here_Background_Reading/Multi-process_Architecture.md)
  
  **Note**: Most of the rest of the design documents assume familiarity with the concepts explained in this document.

  - [How Chromium Displays Web Pages](Start_Here_Background_Reading/How_Chromium_Displays_Web_Pages.md): Bottom-to-top overview of how WebKit is embedded in Chromium
  
##See Also: Design docs in source code

    https://chromium.googlesource.com/chromium/src/+/master/docs/

##General Architecture

- [General Architecture](General_Architecture/README.md)
  - [Conventions and patterns for multi-platform development](General_Architecture/Conventions_and_patterns_for_multi-platform_development.md)
  - [Extension Security Architecture](General_Architecture/Extension_Security_Architecture.md): How the extension system helps reduce the severity of extension vulnerabilities
  - [HW Video Acceleration in Chrom{e,ium}{,OS}](General_Architecture/HW_Video_Acceleration_in_Chrom{eium}{OS}.md)
  - [Inter-process Communication](General_Architecture/Inter-process_Communication.md): How the browser, renderer, and plugin processes communicate
  - [Multi-process Resource Loading](General_Architecture/Multi-process_Resource_Loading.md): How pages and images are loaded from the network into the renderer
  - [Plugin Architecture](General_Architecture/Plugin_Architecture.md)
  - [Process Models](General_Architecture/Process_Models.md): Our strategies for creating new renderer processes
  - [Profile Architecture](General_Architecture/Profile_Architecture.md)
  - [SafeBrowsing](General_Architecture/SafeBrowsing.md)
  - [Sandbox](General_Architecture/Sandbox.md)
  - [Security Architecture](General_Architecture/Security_Architecture.md): How Chromium's sandboxed rendering engine helps protect against malware
  - [Startup](General_Architecture/Startup.md)
  - [Threading](General_Architecture/Threading.md): How to use threads in Chromium
 Also see the documentation for [V8](http://code.google.com/apis/v8/), which is the JavaScript engine used within Chromium.
- [UI Framework](UI_Framework/README.md)
  - [UI Development Practices](UI_Framework/UI_Development_Practices.md): Best practices for UI development inside and outside of Chrome's content areas.
  - [Views framework](UI_Framework/Views_framework.md): Our UI layout layer used on Windows/Chrome OS.
  - [views Windowing system](UI_Framework/views_Windowing_system.md): How to build dialog boxes and other windowed UI using views.
  - [Aura](UI_Framework/Aura.md): Chrome's next generation hardware accelerated UI framework, and the new ChromeOS window manager built using it.
  - [NativeControls](UI_Framework/NativeControls.md): using platform-native widgets in views.
  - Focus and Activation with Views and Aura.
- [Graphics](Graphics/README.md)
  - [Overview](Graphics/Overview.md)
  - [GPU Accelerated Compositing in Chrome](Graphics/GPU_Accelerated_Compositing_in_Chrome.md)
  - [GPU Feature Status Dashboard](Graphics/GPU_Feature_Status_Dashboard.md)
  - [Rendering Architecture Diagrams](Graphics/Rendering_Architecture_Diagrams.md)
  - [Graphics and Skia](Graphics/Graphics_and_Skia.md)
  - [RenderText and Chrome UI text drawing](Graphics/RenderText_and_Chrome_UI_text_drawing.md)
  - [GPU Command Buffer](Graphics/GPU_Command_Buffer.md)
  - [GPU Program Caching](Graphics/GPU_Program_Caching.md)
  - [Compositing in Blink/WebCore](Graphics/Compositing_in_Blink_WebCore.md)
  - [Compositor Thread Architecture](Graphics/Compositor_Thread_Architecture.md)
  - [Rendering Benchmarks](Graphics/Rendering_Benchmarks.md)
  - [Impl-side Painting](Graphics/Impl-side_Painting.md)
  - [Video Playback and Compositor](Graphics/Video_Playback_and_Compositor.md)
  - [ANGLE architecture presentation](Graphics/ANGLE_architecture_presentation.md)
- [Network stack](Network_stack/README.md)
  - [Overview](Network_stack/Overview.md)
  - [Network Stack Objectives](Network_stack/Network_Stack_Objectives.md)
  - [Crypto](Network_stack/Crypto.md)
  - [Disk Cache](Network_stack/Disk_Cache.md)
  - [HTTP Cache](Network_stack/HTTP_Cache.md)
  - [Out of Process Proxy Resolving Draft [unimplemented]](Network_stack/Out_of_Process_Proxy_Resolving_Draft_[unimplemented].md)
  - [Proxy Settings and Fallback](Network_stack/Proxy_Settings_and_Fallback.md)
  - [Debugging network proxy problems](Network_stack/Debugging_network_proxy_problems.md)
  - [HTTP Authentication](Network_stack/HTTP_Authentication.md)
  - [View network internals tool](Network_stack/View_network_internals_tool.md)
  - [Make the web faster with SPDY pages](Network_stack/Make_the_web_faster_with_SPDY_pages.md)
  - [ the web even faster with QUIC pages](Network_stack/_the_web_even_faster_with_QUIC_pages.md)
  - [Cookie storage and retrieval](Network_stack/Cookie_storage_and_retrieval.md)
- [Security](Security/README.md)
  - [Security Overview](Security/Security_Overview.md)
  - [Protecting Cached User Data](Security/Protecting_Cached_User_Data.md)
  - [System Hardening](Security/System_Hardening.md)
  - [Chaps Technical Design](Security/Chaps_Technical_Design.md)
  - [TPM Usage](Security/TPM_Usage.md)
  - [Per-page Suborigins](Security/Per-page_Suborigins.md)
  - [Encrypted Partition Recovery](Security/Encrypted_Partition_Recovery.md)
- [Input](Input/README.md)
  - See[chromium input](Input/chromium_input.md)for design docs and other resources.
- [Rendering](Rendering/README.md)
  - [Multi-column layout](Rendering/Multi-column_layout.md)
  - [Style Invalidation in Blink](Rendering/Style_Invalidation_in_Blink.md)
  - [Blink Coordinate Spaces](Rendering/Blink_Coordinate_Spaces.md)
- [Building](Building/README.md)
  - [IDL build](Building/IDL_build.md)
  - [IDL compiler](Building/IDL_compiler.md)
  - See also the documentation for [GYP, the build script generation tool.](Building/GYP_the_build_script_generation_tool..md)
- [Testing](Testing/README.md)
  - [Layout test results dashboard](Testing/Layout_test_results_dashboard.md)
  - [Generic theme for Test Shell](Testing/Generic_theme_for_Test_Shell.md)
  - [Moving LayoutTests fully upstream](Testing/Moving_LayoutTests_fully_upstream.md)
- [Feature-Specific](Feature-Specific/README.md)
  - [about:conflicts](Feature-Specific/aboutconflicts.md)
  - [Accessibility](Feature-Specific/Accessibility.md): An outline of current (and coming) accessibility support.
  - [Auto-Throttled Screen Capture and Mirroring](Feature-Specific/Auto-Throttled_Screen_Capture_and_Mirroring.md)
  - [Browser Window](Feature-Specific/Browser_Window.md)
  - [Chromium Print Proxy](Feature-Specific/Chromium_Print_Proxy.md): Enables a cloud print service for legacy printers and future cloud-aware printers.
  - [Constrained Popup Windows](Feature-Specific/Constrained_Popup_Windows.md)
  - [Desktop Notifications](Feature-Specific/Desktop_Notifications.md)
  - [DirectWrite Font Cache for Chrome on Windows](Feature-Specific/DirectWrite_Font_Cache_for_Chrome_on_Windows.md)
  - [DNS Prefetching](Feature-Specific/DNS_Prefetching.md): Reducing perceived latency by resolving domain names before a user tries to follow a link
  - [Embedding Flash Fullscreen in the Browser Window](Feature-Specific/Embedding_Flash_Fullscreen_in_the_Browser_Window.md)
  - [Extensions: Design documents and proposed APIs. ](Feature-Specific/Extensions_Design_documents_and_proposed_APIs..md): Design documents and proposed APIs.
  - [Find Bar](Feature-Specific/Find_Bar.md)
  - [Form Autofill](Feature-Specific/Form_Autofill.md): A feature to automatically fill out an html form with appropriate data.
  - [Geolocation](Feature-Specific/Geolocation.md): Adding support for [W3C Geolocation API](http://www.w3.org/TR/geolocation-API/) using native WebKit bindings.
  - [IDN in Google Chrome](Feature-Specific/IDN_in_Google_Chrome.md)
  - [IndexedDB (early draft)](Feature-Specific/IndexedDB__early_draft_.md)
  - [Info Bars](Feature-Specific/Info_Bars.md)
  - [Installer](Feature-Specific/Installer.md): Registry entries and shortcuts
  - [Instant](Feature-Specific/Instant.md)
  - [Isolated Sites](Feature-Specific/Isolated_Sites.md)
  - [Linux Resources and Localized Strings](Feature-Specific/Linux_Resources_and_Localized_Strings.md): Loading data resources and localized strings on Linux.
  - [Media Router & Web Presentation API](Feature-Specific/Media_Router_&_Web_Presentation_API.md)
  - [Memory Usage Backgrounder](Feature-Specific/Memory_Usage_Backgrounder.md): Some information on how we measure memory in Chromium.
  - [Mouse Lock](Feature-Specific/Mouse_Lock.md)
  - [Omnibox Autocomplete](Feature-Specific/Omnibox_Autocomplete/README.md): While typing into the omnibox, Chromium searches for and suggests possible completions.
    - [HistoryQuickProvider](Feature-Specific/Omnibox_Autocomplete/HistoryQuickProvider.md): Suggests completions from the user's historical site visits.
  - [Omnibox/IME Coordination](Feature-Specific/Omnibox_IME_Coordination.md)
  - [Ozone Porting Abstraction](Feature-Specific/Ozone_Porting_Abstraction.md)
  - [Password Generation](Feature-Specific/Password_Generation.md)
  - [Pepper plugin implementation](Feature-Specific/Pepper_plugin_implementation.md)
  - [Plugin Power Saver](Feature-Specific/Plugin_Power_Saver.md)
  - [Preferences](Feature-Specific/Preferences.md)
  - [Prerender](Feature-Specific/Prerender.md)
  - [Print Preview](Feature-Specific/Print_Preview.md)
  - [Printing](Feature-Specific/Printing.md)
  - [Rect-based event targeting in views](Feature-Specific/Rect-based_event_targeting_in_views.md): Making it easier to target views elements with touch.
  - [Replace the modal cookie prompt](Feature-Specific/Replace_the_modal_cookie_prompt.md)
  - [SafeSearch](Feature-Specific/SafeSearch.md)
  - [Sane Time](Feature-Specific/Sane_Time.md): Determining an accurate time in Chrome
  - [Secure Web Proxy](Feature-Specific/Secure_Web_Proxy.md)
  - [Service Processes](Feature-Specific/Service_Processes.md)
  - [Site Isolation](Feature-Specific/Site_Isolation.md): In-progress effort to improve Chromium's process model for security between web sites.
  - [Software Updates: Courgette](Feature-Specific/Software_Updates_Courgette.md)
  - [Sync](Feature-Specific/Sync.md)
  - [Tab Helpers](Feature-Specific/Tab_Helpers.md)
  - [Tab to search](Feature-Specific/Tab_to_search.md): How to have the Omnibox automatically provide tab to search for your site.
  - [Tabtastic2 Requirements](Feature-Specific/Tabtastic2_Requirements.md)
  - [Temporary downloads](Feature-Specific/Temporary_downloads.md)
  - [Time Sources](Feature-Specific/Time_Sources.md): Determining the time on a Chrome OS device
  - [TimeTicks](Feature-Specific/TimeTicks.md): How our monotonic timer, TimeTicks, works on different OSes
  - [UI Mirroring Infrastructure](Feature-Specific/UI_Mirroring_Infrastructure.md): Describes the UI framework in ChromeViews that allows mirroring the browser UI in RTL locales such as Hebrew and Arabic.
  - [UI Localization](Feature-Specific/UI_Localization.md): Describes how localized strings get added to Chromium.
  - [User scripts](Feature-Specific/User_scripts.md): Information on Chromium's support for user scripts.
  - [Video](Feature-Specific/Video.md)
  - [WebSocket](Feature-Specific/WebSocket.md): Enables Web applications to maintain bidirectional communications with server-side processes.
  - [Web MIDI](Feature-Specific/Web_MIDI.md)
  - [WebNavigation API internals](Feature-Specific/WebNavigation_API_internals.md)
- [OS-Specific](OS-Specific/README.md)
  - [Android](OS-Specific/Android/README.md)
    - [Java Resources on Android](OS-Specific/Android/Java_Resources_on_Android.md)
    - [JNI Bindings](OS-Specific/Android/JNI_Bindings.md)
    - [WebView code organization](OS-Specific/Android/WebView_code_organization.md)
  - [Chrome OS](OS-Specific/Chrome_OS/README.md)
    - See the [Chrome OS design documents section.](OS-Specific/Chrome_OS/Chrome_OS_design_documents_section..md)section.
  - [Mac OS X](OS-Specific/Mac_OS_X/README.md)
    - [AppleScript Support](OS-Specific/Mac_OS_X/AppleScript_Support.md)
    - [BrowserWindowController Object Ownership](OS-Specific/Mac_OS_X/BrowserWindowController_Object_Ownership.md)
    - [Confirm to Quit](OS-Specific/Mac_OS_X/Confirm_to_Quit.md)
    - [Mac App Mode (Draft)](OS-Specific/Mac_OS_X/Mac_App_Mode__Draft_.md)
    - [Mac Fullscreen Mode (Draft)](OS-Specific/Mac_OS_X/Mac_Fullscreen_Mode__Draft_.md)
    - [Mac NPAPI Plugin Hosting](OS-Specific/Mac_OS_X/Mac_NPAPI_Plugin_Hosting.md)
    - [Mac specific notes on UI Localization](OS-Specific/Mac_OS_X/Mac_specific_notes_on_UI_Localization.md)
    - [Menus, Hotkeys, & Command Dispatch](OS-Specific/Mac_OS_X/Menus_Hotkeys_&_Command_Dispatch.md)
    - [Notes from meeting on IOSurface usage and semantics](OS-Specific/Mac_OS_X/Notes_from_meeting_on_IOSurface_usage_and_semantics.md)
    - [OS X Interprocess Communication (Obsolete)](OS-Specific/Mac_OS_X/OS_X_Interprocess_Communication__Obsolete_.md)
    - [Password Manager/Keychain Integration](OS-Specific/Mac_OS_X/Password_Manager_Keychain_Integration.md)
    - [Sandboxing Design](OS-Specific/Mac_OS_X/Sandboxing_Design.md)
    - [Tab Strip Design (Includes tab layout and tab dragging)](OS-Specific/Mac_OS_X/Tab_Strip_Design__Includes_tab_layout_and_tab_dragging_.md)
    - [Wrench Menu Buttons](OS-Specific/Mac_OS_X/Wrench_Menu_Buttons.md)
- [Other](Other/README.md)
  - [64-bit Support](Other/64-bit_Support.md)
  - [Browser Components / Layered Components](Other/Browser_Components___Layered_Components.md)
  - [Closure Compiling Chrome Code](Other/Closure_Compiling_Chrome_Code.md)
  - [content module / content API](Other/content_module___content_API.md)
  - [Design docs that still need to be written (wiki)](Other/Design_docs_that_still_need_to_be_written__wiki_.md)
  - [In progress refactoring of key browser-process architecture for porting](Other/In_progress_refactoring_of_key_browser-process_architecture_for_porting.md)
  - [Network Experiments](Other/Network_Experiments.md)
  - [Transitioning InlineBoxes from floats to LayoutUnits](Other/Transitioning_InlineBoxes_from_floats_to_LayoutUnits.md)

