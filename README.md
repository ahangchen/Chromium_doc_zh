# Chromium_doc_zh
Chromium中文文档

https://www.chromium.org/developers/design-documents

翻译之加强对android webview理解

##Design Documents

- Start Here: Background Reading

 - Multi-process Architecture: Describes the high-level architecture of Chromium 

   Note: Most of the rest of the design documents assume familiarity with the concepts explained in this document.

 - How Chromium Displays Web Pages: Bottom-to-top overview of how WebKit is embedded in Chromium

##See Also: Design docs in source code

- https://chromium.googlesource.com/chromium/src/+/master/docs/

##General Architecture

Conventions and patterns for multi-platform development

Extension Security Architecture: How the extension system helps reduce the severity of extension vulnerabilities

HW Video Acceleration in Chrom{e,ium}{,OS}

Inter-process Communication: How the browser, renderer, and plugin processes communicate

Multi-process Resource Loading: How pages and images are loaded from the network into the renderer

Plugin Architecture

Process Models: Our strategies for creating new renderer processes

Profile Architecture

SafeBrowsing

Sandbox

Security Architecture: How Chromium's sandboxed rendering engine helps protect against malware

Startup

Threading: How to use threads in Chromium

Also see the documentation for V8, which is the JavaScript engine used within Chromium.


UI Framework

UI Development Practices: Best practices for UI development inside and outside of Chrome's content areas.

Views framework: Our UI layout layer used on Windows/Chrome OS.

views Windowing system: How to build dialog boxes and other windowed UI using views.

Aura: Chrome's next generation hardware accelerated UI framework, and the new ChromeOS window manager built using it.

NativeControls: using platform-native widgets in views.
Focus and Activation with Views and Aura.

Graphics

Overview

GPU Accelerated Compositing in Chrome

GPU Feature Status Dashboard

Rendering Architecture Diagrams

Graphics and Skia

RenderText and Chrome UI text drawing

GPU Command Buffer

GPU Program Caching

Compositing in Blink/WebCore: from WebCore::RenderLayer to cc::Layer

Compositor Thread Architecture

Rendering Benchmarks

Impl-side Painting

Video Playback and Compositor

ANGLE architecture presentation



Network stack

Overview

Network Stack Objectives

Crypto

Disk Cache

HTTP Cache

Out of Process Proxy Resolving Draft [unimplemented]

Proxy Settings and Fallback

Debugging network proxy problems

HTTP Authentication

View network internals tool

Make the web faster with SPDY pages

Make the web even faster with QUIC pages

Cookie storage and retrieval

Security

Security Overview

Protecting Cached User Data

System Hardening

Chaps Technical Design

TPM Usage

Per-page Suborigins

Encrypted Partition Recovery

Input

See chromium input for design docs and other resources.

Rendering

Multi-column layout

Style Invalidation in Blink

Blink Coordinate Spaces

Building

IDL build

IDL compiler

See also the documentation for GYP, which is the build script generation tool.


Testing

Layout test results dashboard

Generic theme for Test Shell

Moving LayoutTests fully upstream

Feature-Specific

about:conflicts

Accessibility: An outline of current (and coming) accessibility support.
Auto-Throttled Screen Capture and Mirroring

Browser Window

Chromium Print Proxy: Enables a cloud print service for legacy printers and future cloud-aware printers.

Constrained Popup Windows

Desktop Notifications

DirectWrite Font Cache for Chrome on Windows

DNS Prefetching: Reducing perceived latency by resolving domain names before a user tries to follow a link

Embedding Flash Fullscreen in the Browser Window

Extensions: Design documents and proposed APIs.


Find Bar

Form Autofill: A feature to automatically fill out an html form with appropriate data.

Geolocation: Adding support for W3C Geolocation API using native WebKit bindings.

IDN in Google Chrome

IndexedDB (early draft)

Info Bars

Installer: Registry entries and shortcuts

Instant

Isolated Sites

Linux Resources and Localized Strings: Loading data resources and localized strings on Linux.

Media Router & Web Presentation API

Memory Usage Backgrounder: Some information on how we measure memory in Chromium.

Mouse Lock

Omnibox Autocomplete: While typing into the omnibox, Chromium searches for and suggests possible completions.

HistoryQuickProvider: Suggests completions from the user's historical site visits.

Omnibox/IME Coordination

Ozone Porting Abstraction

Password Generation

Pepper plugin implementation

Plugin Power Saver

Preferences

Prerender

Print Preview

Printing

Rect-based event targeting in views: Making it easier to target views elements with touch.

Replace the modal cookie prompt

SafeSearch

Sane Time: Determining an accurate time in Chrome

Secure Web Proxy

Service Processes

Site Isolation: In-progress effort to improve Chromium's process model for security between web sites.

Software Updates: Courgette

Sync

Tab Helpers

Tab to search: How to have the Omnibox automatically provide tab to search for your site.

Tabtastic2 Requirements

Temporary downloads

Time Sources: Determining the time on a Chrome OS device

TimeTicks: How our monotonic timer, TimeTicks, works on different OSes
UI Mirroring Infrastructure: Describes the UI framework in ChromeViews that allows mirroring the browser UI in RTL locales such as Hebrew and Arabic.

UI Localization: Describes how localized strings get added to Chromium.
User scripts: Information on Chromium's support for user scripts.

Video

WebSocket: Enables Web applications to maintain bidirectional communications with server-side processes.

Web MIDI

WebNavigation API internals



OS-Specific

Android

Java Resources on Android

JNI Bindings

WebView code organization

Chrome OS

See the Chrome OS design documents section.

Mac OS X

AppleScript Support

BrowserWindowController Object Ownership

Confirm to Quit

Mac App Mode (Draft)

Mac Fullscreen Mode (Draft)

Mac NPAPI Plugin Hosting

Mac specific notes on UI Localization

Menus, Hotkeys, & Command Dispatch

Notes from meeting on IOSurface usage and semantics

OS X Interprocess Communication (Obsolete)

Password Manager/Keychain Integration

Sandboxing Design

Tab Strip Design (Includes tab layout and tab dragging)

Wrench Menu Buttons



Other

64-bit Support

Browser Components / Layered Components

Closure Compiling Chrome Code

content module / content API

Design docs that still need to be written (wiki)

In progress refactoring of key browser-process architecture for porting

Network Experiments

Transitioning InlineBoxes from floats to LayoutUnits

