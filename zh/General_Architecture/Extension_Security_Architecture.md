# Web安全研究
## 保护浏览器不受扩展的缺陷影响

[保护浏览器不受扩展的缺陷影响](http://www.eecs.berkeley.edu/Pubs/TechRpts/2009/EECS-2009-185.pdf)

[Adam Barth](http://www.adambarth.com/), [Adrienne Porter Felt](http://www.eecs.berkeley.edu/~afelt/), [Prateek Saxena](http://www.cs.berkeley.edu/~prateeks/), and [Aaron Boodman](http://www.aaronboodman.com/)

*EECS Department. University of California, Berkeley. Technical Report No. UCB/EECS-2009-185*
### 摘要

浏览器扩展非常流行，三分之一的Firefox用户运行至少一个扩展。尽管出于好意，扩展的开发者通常不是安全专家，并且会写有bug的代码，而这可能被网站操作者利用。在Firefox扩展系统中，这种利用很危险，因为扩展运行时拥有用户的完全权限，可以读写任意文件，启动新的进程。在这篇文章里，我们分析了25个最受欢迎的火狐扩展，并且发现这些扩展中的88%不需要完整的可用权限。另外，我们发现这些扩展中的76%不必要地使用了功能强大的api，使得降低他们的权限变得困难。我们提出一个新的浏览器扩展系统，通过使用最少的权限，权限分割，强解耦，提高安全性。我们的系统限制了一个攻击者通过扩展的缺陷所能做到的罪行。我们的设计被Google Chrome扩展系统接受。


这篇文章的一个扩展版本可以在Proc. of the 17th Network and Distributed System Security Symposium (NDSS 2010)看到。

[更多Berkeley web安全研究>>](http://webblaze.cs.berkeley.edu/)