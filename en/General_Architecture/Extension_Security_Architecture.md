# Web Security Research
## Protecting Browsers from Extension Vulnerabilities

[Protecting Browsers from Extension Vulnerabilities](http://www.eecs.berkeley.edu/Pubs/TechRpts/2009/EECS-2009-185.pdf)

[Adam Barth](http://www.adambarth.com/), [Adrienne Porter Felt](http://www.eecs.berkeley.edu/~afelt/), [Prateek Saxena](http://www.cs.berkeley.edu/~prateeks/), and [Aaron Boodman](http://www.aaronboodman.com/)

*EECS Department. University of California, Berkeley. Technical Report No. UCB/EECS-2009-185*
### Abstract

Browser extensions are remarkably popular, with one in three Firefox users running at least one extension. Although well-intentioned, extension developers are often not security experts and write buggy code that can be exploited by malicious web site operators. In the Firefox extension system, these exploits are dangerous because extensions run with the user's full privileges and can read and write arbitrary files and launch new processes. In this paper, we analyze 25 popular Firefox extensions and find that 88% of these extensions need less than the full set of available privileges. Additionally, we find that 76% of these extensions use unnecessarily powerful APIs, making it difficult to reduce their privileges. We propose a new browser extension system that improves security by using least privilege, privilege separation, and strong isolation. Our system limits the misdeeds an attacker can perform through an extension vulnerability. Our design has been adopted as the Google Chrome extension system.

An extended version of this paper will appear at Proc. of the 17th Network and Distributed System Security Symposium (NDSS 2010).

[More Berkeley web security research >>](http://webblaze.cs.berkeley.edu/)