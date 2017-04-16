# Chromium浏览器安全架构

大部分当前web浏览器使用一种将用户和网络结合成一个单一保护域的单片结构。在这样的浏览器中，攻击者可以利用任意代码执行漏洞，盗取敏感信息或者安装恶意软件。在这篇文章里，我们展示了Chromium的安全架构。Chromium有着两个处于独立保护域 的模块：一个是浏览器内核，与操作系统交互，一个是渲染引擎，运行在只有限制权限的沙箱中。这种架构有助于减少高危攻击，而不牺牲与现有网站的兼容性。我们为浏览器漏洞定义了一个威胁模型，并评估了这种架构会如何减少过去的问题。


**[The Security Architecture of the Chromium Browser](http://seclab.stanford.edu/websec/chromium/chromium-security-architecture.pdf)**

[Adam Barth](http://www.adambarth.com/), [Collin Jackson](http://www.collinjackson.com/), [Charles Reis](http://www.cs.washington.edu/homes/creis/), and [The Google Chrome Team](http://www.google.com/chrome)
Technical Report 2008

[More Stanford web security research](http://seclab.stanford.edu/websec/)