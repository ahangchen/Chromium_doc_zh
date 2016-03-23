#进程模型

这个文档描述了Chromium支持的不同线程模型，包括它的渲染器进程，以及现有模型实现的问题。

##概述

网页内容已经发展到包含大量在浏览器内运行的活跃代码的地步，使得许多网站更像应用程序而非文档。这种变革改变了浏览器的角色，从一个简单的文档渲染器变成一个操作系统。Chromium构建得像一个操作系统那样，使用多进程隔离每个网站和浏览器自身，以一种安全而鲁棒的方式运行这些程序。这提高了鲁棒性，因为每个进程运行在自己的地址空间里，由操作系统调度，即使崩溃也不会互相影响。用户也可以在Chromium的任务管理器里查看每个进程的资源使用情况。

Web浏览器有许多方法可以分割成不同的操作系统进程，最佳的架构的选择取决于许多因素，包括稳定性，资源使用，对实际情况的观察。Chromium支持四种不同的进程模型，允许开发者实验，也有最适合大部分用户的默认模式。


##支持的模型

Chromium支持四种不同的模型，它们影响浏览器分配页面给渲染进程的行为。默认情况下，Chromium为用户访问的每个网站使用一个独立的操作系统进程。然而，用户可以在启动Chromium时指定命令行选项，以选择其他的架构：全网站单进程，每组相连标签页一个进程，或者每个东西都放在一个单独的进程中。这些模型的区别在于他们是否影响内容的源，是否影响标签页间的关系，或者两者都会影响。这个章节在更深的细节上讨论每种模型，并在这个文档的后面描述当前Chromium的实现的一些问题。

###每个网站实例一个进程

默认情况情况下，Chromium为用户访问的每个网站实例创建一个渲染器进程。这保证了不同网站的网页独立渲染，让对同一个网站的不同访问相互独立。
By default, Chromium creates a renderer process for each instance of a site the user visits. This ensures that pages from different sites are rendered independently, and that separate visits to the same site are also isolated from each other. Thus, failures (e.g., renderer crashes) or heavy resource usage in one instance of a site will not affect the rest of the browser. This model is based on both the origin of the content and relationships between tabs that might script each other. As a result, two tabs may display pages that are rendered in the same process, while navigating to a cross-site page in a given tab may switch the tab's rendering process. (Note that there are important caveats in Chromium's current implementation, discussed in the Caveats section below.)

Concretely, we define a "site" as a registered domain name (e.g., google.com or bbc.co.uk) plus a scheme (e.g., https://). This is similar to the origin defined by the Same Origin Policy, but it groups subdomains (e.g., mail.google.com and docs.google.com) and ports (e.g., http://foo.com:8080) into the same site. This is necessary to allow pages that are in different subdomains or ports of a site to access each other via Javascript, which is permitted by the Same Origin Policy if they set their document.domain variables to be identical.

A "site instance" is a collection of connected pages from the same site. We consider two pages as connected if they can obtain references to each other in script code (e.g., if one page opened the other in a new window using Javascript).

**Strengths**

- Isolates content from different sites. This provides a meaningful form of fate sharing for web content, where pages are isolated from failures caused by other web sites.
- Isolates independent tabs showing the same site. Visiting the same site independently in different tabs will create different processes. This will prevent contention and failures in one instance from affecting other instances.
- 
**Weaknesses**

- More memory overhead. In most workloads, this model will create more renderer processes than the process-per-site model described below. While this increases stability and may add opportunities for parallelism, it also increases memory overhead.
- More complex to implement.  Unlike process-per-tab and single-process, this model requires complex logic to support swapping processes in a tab when it navigates between sites, as well as proxying a small set of JavaScript actions that are permitted between origins, such as postMessage.  (For more on this issue and our ongoing efforts to support it fully, see the Caveats section below and our Site Isolation project page.)

###Process-per-site

Chromium also supports a process model that isolates different sites from each other, but groups all instances of the same site into the same process. To use this model, users should specify a --process-per-site command-line switch when starting Chromium. This creates fewer renderer processes, trading some robustness for lower memory overhead. This model is based on the origin of the content and not the relationships between tabs.

**Strengths**

- Isolates content from different sites. As in the process-per-site-instance model, pages from different sites will not share fate.
- Less memory overhead. This model is likely to create fewer concurrent processes than the process-per-site-instance and process-per-tab models. This may be desirable to reduce Chromium's memory footprint.

**Weaknesses**

- Can result in large renderer processes. Sites like google.com host a wide variety of applications that may be open concurrently in the browser, all of which would be rendered in the same process. Thus, resource contention and failures in these applications could affect many tabs, making the browser seem less responsive. It is unfortunately hard to identify site boundaries at a finer granularity than the registered domain name without breaking backwards compatibility.
- More complex to implement.  Like the process-per-site-instance model, this requires logic for swapping processes during navigation and proxying some JavaScript interactions.

###Process-per-tab

The process-per-site-instance and process-per-site models both consider the origin of the content when creating renderer processes. Chromium also supports a simpler model which dedicates one renderer process to each group of script-connected tabs. This model can be selected using the --process-per-tab command-line switch.

Specifically, we refer to a set of tabs with script connections to each other as a browsing instance, which also corresponds to a "unit of related browsing contexts" from the HTML5 spec. This set consists of a tab and any other tabs that it opens using Javascript code. Such tabs must be rendered in the same process to allow Javascript calls to be made between them (most commonly between pages from the same origin).

**Strengths**

- Simple to understand. Each tab has one renderer process dedicated to it that does not change over time.

**Weaknesses**

- Leads to undesirable fate sharing between pages. If the user navigates a tab in a browsing instance to a different web site, the new page will share fate with any other pages in the browsing instance.

It is worth noting that Chromium still forces process swaps within a tab in some situations in the process-per-tab model, when it is required for security.  For example, normal web pages are not allowed to share a process with privileged pages like Settings and the New Tab Page.  As a result, this model is not significantly simpler in practice than process-per-site-instance.

###Single process

Finally, for the purposes of comparison, Chromium supports a single process model that can be enabled using the --single-process command-line switch. In this model, both the browser and rendering engine are run within a single OS process.

The single process model provides a baseline for measuring any overhead that the multi-process architectures impose. It is not a safe or robust architecture, as any renderer crash will cause the loss of the entire browser process. It is designed for testing and development purposes, and it may contain bugs that are not present in the other architectures.

##Sandboxes and plug-ins

In each of the multi-process architectures, Chromium's renderer processes are executed within a sandboxed process that has limited access to the user's computer. These processes do not have direct access to the user's filesystem, display, or most other resources. Instead, they gain access to permitted resources only through the browser process, which can impose security policies on this access. As a result, Chromium's browser process can mitigate the damage that an exploited rendering engine can do.

Browser plug-ins, such as Flash and Silverlight, are also executed in their own processes, and some plug-ins like Flash even run within Chromium's sandbox. In each of the multi-process architectures that Chromium supports, there is one process instance for each type of active plug-in. Thus, all Flash instances run in the same process, regardless of which sites or tabs they appear in.

##Caveats

This section lists a few caveats with Chromium's current implementation of the process models, along with their implications.

- Most renderer-initiated navigations within a tab do not yet lead to process swaps. If the user follows a link, submits a form, or is redirected by a script, Chromium will not attempt to switch renderer processes in the tab if the navigation is cross-site. Chromium only swaps renderer processes for browser-initiated cross-site navigations, such as typing a URL in the location bar or following a bookmark. As a result, pages from different sites may be rendered in the same process, even in the process-per-site-instance and process-per-site models. This is likely to change in future versions of Chromium as part of the Site Isolation project.

However, there is a mechanism web pages can use to suggest that a link points to an unrelated page and can be safely rendered in a different process.  If a link has the rel=noreferrer target=_blank attributes, then Chromium will typically render it in a different process.

- Subframes are currently rendered in the same process as their parent page. Although cross-site subframes do not have script access to their parents and could safely be rendered in a separate process, Chromium does not yet render them in their own processes. Similar to the first caveat, this means that pages from different sites may be rendered in the same process. This will likely change in future versions of Chromium.

- There is a limit to the number of renderer processes that Chromium will create. This prevents the browser from overwhelming the user's computer with too many processes. The limit is is proportional to the amount of memory on the computer, and may be as high as 80 processes. Because of the limit, a single renderer process may be dedicated to multiple sites. This reuse is currently done at random, but future versions of Chromium may apply heuristics to more intelligently allocate sites to renderer processes.
##Implementation notes

Two classes in Chromium represent the abstractions needed for the various process models: BrowsingInstance and SiteInstance.

The BrowsingInstance class represents a set of script-connected tabs within the browser, also known as a unit of related browsing contexts in the HTML 5 spec. In the process-per-tab model, we create a renderer process for each BrowsingInstance.

The SiteInstance class represents a set of connected pages from the same site. It is a subdivision of pages within a BrowsingInstance, and it is important that there is only one SiteInstance per site within a BrowsingInstance. In the process-per-site-instance model, we create a renderer process for each SiteInstance. To implement process-per-site, we ensure that all SiteInstances from the same site end up in the same process.

##Academic Papers

[Isolating Web Programs in Modern Browser Architectures](http://www.charlesreis.com/research/publications/eurosys-2009.pdf)

Charles Reis, Steven D. Gribble  (both authors at UW + Google)

Eurosys 2009. Nuremberg, Germany, April 2009.

Abstract:

*Many of today's web sites contain substantial amounts of client-side code, and consequently, they act more like programs than simple documents. This creates robustness and performance challenges for web browsers. To give users a robust and responsive platform, the browser must identify program boundaries and provide isolation between them.*

*We provide three contributions in this paper. First, we present abstractions of web programs and program instances, and we show that these abstractions clarify how browser components interact and how appropriate program boundaries can be identified. Second, we identify backwards compatibility tradeoffs that constrain how web content can be divided into programs without disrupting existing web sites. Third, we present a multi-process browser architecture that isolates these web program instances from each other, improving fault tolerance, resource management, and performance. We discuss how this architecture is implemented in Google Chrome, and we provide a quantitative performance evaluation examining its benefits and costs.*

[Security Architecture of the Chromium Browser](http://crypto.stanford.edu/websec/chromium/)
Adam Barth, Collin Jackson, Charles Reis, and The Google Chrome Team

Stanford Technical Report, September 2008. 

Abstract:

*Most current web browsers employ a monolithic architecture that combines "the user" and "the web" into a single protection domain. An attacker who exploits an arbitrary code execution vulnerability in such a browser can steal sensitive files or install malware. In this paper, we present the security architecture of Chromium, the open-source browser upon which Google Chrome is built. Chromium has two modules in separate protection domains: a browser kernel, which interacts with the operating system, and a rendering engine, which runs with restricted privileges in a sandbox. This architecture helps mitigate high-severity attacks without sacrificing compatibility with existing web sites. We define a threat model for browser exploits and evaluate how the architecture would have mitigated past vulnerabilities.*