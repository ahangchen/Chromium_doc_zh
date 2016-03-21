#插件架构
##背景
在阅读这个文档前，你应当熟悉Chromium的多进程架构。

##概述

插件是浏览器不稳定的主要来源。插件也会在渲染器没有实际运行时，让进程沙箱化。因为进程是第三方编写的，我们无法控制他们对操作系统的访问。解决方案是，让插件在各自独立的进程中运行。


##设计细节

###进程内插件

Chromium有着在进程内运行插件的能力（对测试来讲非常方便），也可以在进程外运行插件。它们都始于我们的非多进程WebKit嵌入层，期待嵌入层实现WebKit::WebPlugin接口。这实际由WebPluginImpl实现。WebPluginImpl在图中的虚线以上，与WebPluginDelegate接口交流，对进程内插件而言，这个接口由WebPluginDelegateImpl实现，它会接着与我们的NPAPI包装层通信。



![](../in_process_plugins.png)

历史经验：还没有WebKit嵌入层的时候，WebPluginImpl是对应的嵌入层。它会与“嵌入应用程序”通过WebPluginDelegate抽象接口交流，我们通过切换这个接口的实现，服务与进程内插件与进程外插件。在有了额外的Chromium WebKit API之后，我们增加了新的WebKit::WebPlugin抽象接口，它与旧的WebPluginDelegate接口有着相同的功能。这个接口的一个好一点的设计是，合并WebPluginImpl和WebPluginDelegateImpl，在WebKit::WebPlugin层做进程划分。由于这个问题的复杂性，现在还没有这样实现。


###进程外插件

Chromium通过切换上面的图中，虚线以上几层的实现来支持跨进程插件。这干预了WebPluginImpl层和WebPluginDelegateImpl之间的IPC层，并让我们在每个模式之间共享我们所有的NPAPI代码。所有旧的WebPluginDelegateImpl代码，以及与它通信的NPAPI层，现在是在独立的插件进程中执行了。


The two sides of a Renderer/Plugin communication channel are represented by the PluginChannel and the PluginChannelHost. We have many renderer processes, and one plugin process for each unique plugin. This means there is one PluginChannelHost in the renderer for each type of plugin it uses (for example, Adobe Flash and Windows Media Player). In each plugin process, there will be one PluginChannel for each renderer process that has an instance of that type of plugin.

Each end of the channel, in turn, maps to many different instances of a plugin. For example, if there are two Adobe Flash movies embedded in a web page, there will be two WebPluginDelegateProxies (and related stuff) in the renderer side, and two WebPluginDelegateStubs (and related stuff)
in the plugin side. The channel is in charge of multiplexing the communication between these many objects over one IPC connection.

In this diagram, you can see the classes from the above in-process diagram grayed out, with the new out-of-process layer (in color) in the middle.

![](../out_of_process_plugins.png)

**Historical note**: We used to consistently use a stub/proxy model for communication, with one stub and one proxy on each side of the IPC channel for receiving and sending messages to one plugin, respectively. This leads to many classes that can get confusing. As a result, the WebPluginStub was merged into the WebPluginDelegateProxy which now handles the renderer side of all IPC communication for one plugin instance. The plugin side has not been merged yet, leaving two classes, the WebPluginDelegateStub and the WebPluginProxy as conceptually the same object, just representing different directions of communication.

###Windowless Plugins

Windowless plugins are designed to run directly within the rendering pipeline.  When WebKit wants to draw a region of the screen involving the plugin it calls into the plugin code, handing it a drawing context.  Windowless plugins are often used in situations where the plugin is expected to be transparent over the page -- it's up to the plugin drawing code to decide how it munges the bit of the page it's given.

To take windowless plugins out of process, you still need to incorporate their rendering in the (synchronous) rendering pass done by WebKit.  A naively slow option is to clip out the region that the plugin will draw on then synchronously ship that over to the plugin process and let it draw.  This can then be sped up with some shared memory.

However, rendering speed is then at the mercy of the plugin process (imagine a page with 30 transparent plugins -- we'd need 30 round trips to the plugin process).  So instead we have windowless plugins asynchronously paint, much like how our existing page rendering is asynchronous with respect to the screen.  The renderer has effectively a backing store of what the plugin's rendered area looks like and uses this image when drawing, and the plugin is free to asynchronously send over new updates representing changes to the rendered area.

All of this is complicated a bit by "transparent" plugins.  The plugin process needs to know what pixels it wants to draw over.  So it also keeps a cache of what the renderer last sent it as the page background behind the plugin, then lets the plugin repeatedly draw over that. 

So, in all, here are the buffers involved for the region drawn by a windowless plugin:

Renderer process
 - backing store of what the plugin last rendered as
 - shared memory with the plugin for receiving updates ("transport DIB")
 - copy of the page background behind the plugin (described below)

Plugin process
 - copy of the page background behind the plugin, used as the source
material when drawing
 - shared memory with the renderer for sending updates ("transport DIB")

Why does the renderer keep a copy of the page background?  Because if the page background changes, we need to synchronously get the plugin to redraw over the new background it will appear to lag.  We can tell that the background changed by comparing the newly-rendered background against our copy of what the plugin thinks the background.  Since the plugin and renderer processes are asynchronous with respect to one another, they need separate copies.
###Overall system

This image shows the overall system with the browser and two renderer processes, each communicating with one shared out-of-process Flash process. There are three total plugin instances. Note that this diagram is out of date, and WebPluginStub has been merged with WebPluginDelegateProxy.

![](../pluginsoutofprocess.png)