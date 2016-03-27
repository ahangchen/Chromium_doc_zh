#进程模型

这个文档描述了Chromium支持的不同线程模型，包括它的渲染器进程，以及现有模型实现的问题。

##概述

网页内容已经发展到包含大量在浏览器内运行的活跃代码的地步，使得许多网站更像应用程序而非文档。这种变革改变了浏览器的角色，从一个简单的文档渲染器变成一个操作系统。Chromium构建得像一个操作系统那样，使用多进程隔离每个网站和浏览器自身，以一种安全而鲁棒的方式运行这些程序。这提高了鲁棒性，因为每个进程运行在自己的地址空间里，由操作系统调度，即使崩溃也不会互相影响。用户也可以在Chromium的任务管理器里查看每个进程的资源使用情况。

Web浏览器有许多方法可以分割成不同的操作系统进程，最佳的架构的选择取决于许多因素，包括稳定性，资源使用，对实际情况的观察。Chromium支持四种不同的进程模型，允许开发者实验，也有最适合大部分用户的默认模式。


##支持的模型

Chromium支持四种不同的模型，它们影响浏览器分配页面给渲染进程的行为。默认情况下，Chromium为用户访问的每个网站使用一个独立的操作系统进程。然而，用户可以在启动Chromium时指定命令行选项，以选择其他的架构：全网站单进程，每组相连标签页一个进程，或者每个东西都放在一个单独的进程中。这些模型的区别在于他们是否影响内容的源，是否影响标签页间的关系，或者两者都会影响。这个章节在更深的细节上讨论每种模型，并在这个文档的后面描述当前Chromium的实现的一些问题。

###单网站实例单进程

默认情况下，Chromium为用户访问的每个网站实例创建一个渲染器进程。这保证了不同网站的网页独立渲染，让对同一个网站的不同访问相互独立。因此一个网站实例中的失败（比如，渲染器崩溃）或者重的资源使用不会影响浏览器的其他部分。这个模型基于内容的源和脚本会相互影响的标签页间的关系。因此，两个标签页可以在同一个渲染进程里展示页面，同时在给定的一个标签页中导航到网站外的一个网页，可能切换标签页的渲染进程。（注意，Chromium当前的实现有一些重要的问题，会在下面的Caveat(警告)部分讨论。）

具体说来，我们把一个注册域名（例如：google.com或bbc.co.uk）加一个scheme（例如：https://）定义为一个网站。这与同源策略定义相似，但它将子域名（比如：mail.google.com和docs.google.com）和端口（比如http://foo.com:8080）合并到同一个网站中。允许网站的不同子域名或端口中的页面通过Javascript访问是有必要的，如果他们的document.domain变量相同的话，同源策略也会这样允许。

一个网站实例是一些相同网站的相连网页的集合。我们这样认为两个页面是相连的：如果他们可以在脚本代码中获取彼此的引用的话（比如：如果一个页面被另一个页面用Javascript在一个新窗口中打开）。


**优点**

- 隔离不同网站的内容。这提供了网页内容的命运共享的一种有意义的形式，在这种形式中，网页间的失败不会相互影响。
- 隔离展示相同网站的独立标签页。在不同的标签页中独立访问同样的网站会创建不同的进程。这可以避免同个实例中的争夺与失败，使其不会影响其他实例。
 
**缺点**
- 更多的内存负载。在大多数工作负载下，这个模型会比下面的每个网站一个进程创建更多渲染器进程。这虽然能增加稳定性并且增加并发的机会，但它也增加了内存负载。
- 更复杂的实现。不像每个标签页一个进程或者单进程，这个模型需要复杂的逻辑以支持标签在网页间导航时的进程交换，以及代理一些允许的源之间的JavaScript行为，比如传递消息。（关于这个话题的更多内容以及我们正在进行的对这种模型的完全支持的努力，查看下面的Caveats(警告)部分以及我们的站点隔离工程页面。）


###单网站单进程

Chromium也支持这样一种进程模式，隔离不同的网站，但将相同网站的所有实例组合到一块。为了使用这个模型，用户需要在启动Chromium时在终端指定 --process-per-site开关。这创建更少的渲染进程，用鲁棒性交换更少的内存占用。这个模型基于内容的源，而非标签页间的关系。


**优点**

- 隔离不同网站的内容。正如每个网站实例一个进程的模型那样，不同网站的页面不会共享命运（不会同生共死。。）。

- 更少的内存占用。这个模型比上一个模型和每个标签一个进程的模型可能创建更少的并行进程。这对于减少Chromium的内存足迹可能是需要的。

**缺点**

- 可能导致更大的渲染进程。像google.com这样的站点上有着大量的应用程序，它们可能在浏览器里被同时打开，并且全部在同一个进程里渲染。因此，这些应用程序中的资源争夺与失败会影响许多标签页，使得浏览器看起来不能更好地响应。不幸的是，在细粒度上而非通过注册域名区分网站边界，并且不影响向后兼容性是很困难的。
- 实现更加复杂。与每个网站实例一个进程的模型相似，这需要在导航中交换进程以及代理一些javascript操作的逻辑。

###单标签页单进程

每个网站或每个网站实例一个进程都需要在创建渲染进程时考虑网站内容的源。Chromium也支持一种简单的模型，将一个渲染器进程分配给每组脚本相关的标签页。这个模型可以使用 --process-per-tab命令行开关来选中。

特别地，我们会把一些脚本相关联的标签页成为一个浏览实例，它也与HTML5范畴中的“一个浏览上下文单元”对应。这个集合由一个标签以及这个标签用javascript代码所打开的标签组合而成。这样的标签必须在同一个进程中渲染，以允许在这些标签页间执行javascript调用（大多数通常发生在同源页面之间）。


**优点**

- 容易理解。每个标签页分配有一个渲染进程，并不会随时间改变。


**缺点**

- 导致我们不想要的页面之间命运共享。如果用户在浏览实例中导航一个标签页到一个不同的网站中，新的页面会和其他在同一个浏览实例中的任何其他标签页共享命运。

某些情况下，尽管处于安全的需要，在这个模型中，Chromium仍然强制标签页中的进程交换已经没有什么价值。例如，通常的网页不允许与优先级高的网页（比如设置，或者新标签页）共享进程。因此，这个模型在实践中并没有比单个网站实例单进程更简单。


###单进程

最后，出于比较的目的，Chromium支持单进程模型，通过--single-process命令行开关打开。在这个模型中，浏览器和渲染引擎跑在同一个操作系统进程里。

单进程模型提供了一个衡量多进程架构带来的负荷的基线。这不是一个安全的架构，也不是一个鲁棒的架构，因为任何渲染器的崩溃会导致整个浏览器进程挂掉。它只是设计用于测试和开发目的，并且可能包含在其他架构中没有的bug。


##沙箱与插件

在每个多进程架构里，Chromium的渲染器进程运行在一个沙箱进程中，它对用户电脑只有有限的访问权限。这些进程对用户的文件系统，显示器，或者大部分其他的资源没有直接的接触。相反，他们只通过浏览器进程获得对允许的资源的访问，而浏览器进程可以在这种访问上附加安全策略。因此，Chromium的浏览器进程可以减轻一个被利用的渲染器引擎能做的事情。

浏览器插件，比如Flash和Silverlight，也在他们自己的进程中执行，并且有些插件，比如Flash运行在Chromium的沙箱中。在Chromium支持的每个多进程架构中，对每种活跃的插件都只有一个进程。因此，所有的Flash实例运行在同一个进程里，不论它们出现在哪个网站或标签页中。


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