# The Security Architecture of the Chromium Browser

Most current web browsers employ a monolithic architecture that combines "the user" and "the web" into a single protection domain. An attacker who exploits an arbitrary code execution vulnerability in such a browser can steal sensitive files or install malware. In this paper, we present the security architecture of Chromium, the open-source browser upon which Google Chrome is built. Chromium has two modules in separate protection domains: a browser kernel, which interacts with the operating system, and a rendering engine, which runs with restricted privileges in a sandbox. This architecture helps mitigate high-severity attacks without sacrificing compatibility with existing web sites. We define a threat model for browser exploits and evaluate how the architecture would have mitigated past vulnerabilities.


**[The Security Architecture of the Chromium Browser](http://seclab.stanford.edu/websec/chromium/chromium-security-architecture.pdf)**

[Adam Barth](http://www.adambarth.com/), [Collin Jackson](http://www.collinjackson.com/), [Charles Reis](http://www.cs.washington.edu/homes/creis/), and [The Google Chrome Team](http://www.google.com/chrome)
Technical Report 2008

[More Stanford web security research](http://seclab.stanford.edu/websec/)