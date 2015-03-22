#IO Concept: Blocking, NonBlocking and Sync, Async
`IO` `Blocking` `Non-Blocking` `Sync` `Async` `阻塞` `非阻塞` `同步` `异步`

##引子
IO操作是程序设计与开发中非常重要的关注点。当人们在讨论IO时，人们在讨论什么？<br/>
阻塞、非阻塞、同步、异步。这4个概念常常被提及，但是常常被混淆。说者讲不清，听者想不明。<br/>
本文尝试用最简单的语言介绍这4个概念，点到即止。

##两个角度的概念
阻塞/非阻塞与同步/异步是两个角度的概念。<br/>
`阻塞/非阻塞`：是指IO调用发起后，调用方（用户线程）的状态，即用户线程是在继续向后执行还是在等待。用户线程是否阻塞。<br/>
`同步/异步`：是指IO调用发起后，调用方（用户线程）是否能够立即得到被调用方（内核线程）返回的结果。用户线程与内核线程是否同步。

##IO模型
两个角度的概念组合出4种IO模型。<br/>
###阻塞、同步IO
IO调用发起后，用户线程阻塞，一直等待内核线程返回调用结果。
###阻塞、异步IO
IO调用发起后，不用等待内核线程返回调用结果，用户线程阻塞，等待内核线程的通知。<br/>
很多人组合出这种类型，并以IO多路复用作为实例。某种程度上我不太认可，因为导致线程阻塞的并不是IO读写操作，而是select操作（信号处理），IO读写操作并没有阻塞。所以，也有人称之为“带有阻塞通知的非阻塞IO”。为了概念完整性，先暂且留在这里。
###非阻塞、同步IO
IO调用发起后，用户线程不阻塞，内核线程立即返回调用结果，这个结果可能是IO数据，也可能是错误码。
###非阻塞、异步IO
IO调用发起后，用户线程不阻塞，内核线程立即返回调用结果，该结果是内核调用已经发起。待内核调用完成后会通知用户线程。<br/>
综上，也有人认为是3种IO模型，即阻塞IO（包括前两种）、非阻塞IO和异步IO，即BIO，NIO和AIO。

##小结
区分IO概念，需要注意以下几点：<br/>
* 阻塞/非阻塞与同步/异步是两个角度的问题，不要混着讨论，比如尝试区分阻塞与同步。<br/>
* 用户线程（用户空间、用户态）和内核线程（内核空间、内核态）是在两个层次，应用开发通常是讨论用户线程的问题，因此阻塞/非阻塞IO指的是用户线程是否阻塞。毕竟IO执行时，用户线程、内核线程，直至硬件，总有一个需要等待资源被满足的。而同步/异步是在讨论用户线程与内核线程间是同步还是异步。
* IO操作和IO结果的信号操作是两个概念，处理IO数据和处理IO信号也是不一样的。

##引申阅读
* 简单明了的说明了阻塞IO、非阻塞同步IO、非阻塞异步IO的含义，同时几篇参考文章都值得阅读，Windows系统。链接：[I/O Concept – Blocking/Non-Blocking VS Sync/Async](http://csliu.com/2009/06/io-concept-blockingnon-blocking-vs-syncasync/ "I/O Concept – Blocking/Non-Blocking VS Sync/Async")
* 通过2张图说明了同步IO和异步IO用户态和内核态的工作时序和内容，Windows系统。链接：[Synchronous and Asynchronous I/O](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365683(v=vs.85).aspx "Synchronous and Asynchronous I/O")
* 答主认为只有两种IO：同步IO和异步IO，Java环境。链接：[non-blocking IO vs async IO and implementation in Java](http://stackoverflow.com/questions/25099640/non-blocking-io-vs-async-io-and-implementation-in-java "non-blocking IO vs async IO and implementation in Java")
* libevent的异步IO简介，Linux系统。链接：[A tiny introduction to asynchronous IO](http://www.wangafu.net/~nickm/libevent-book/01_intro.html "A tiny introduction to asynchronous IO")
* 通过5张图说明了阻塞IO、非阻塞IO、IO复用、信号驱动异步IO、异步IO用户态和内核态的工作时序和内容，Linux系统。链接：[linux基础编程：IO模型：阻塞/非阻塞/IO复用 同步/异步 Select/Epoll/AIO](http://blog.csdn.net/colzer/article/details/8169075 "linux基础编程：IO模型：阻塞/非阻塞/IO复用 同步/异步 Select/Epoll/AIO")
* 说明同步异步、阻塞与非阻塞的概念差异。链接：[同步与异步、阻塞与非阻塞](http://www.cnblogs.com/albert1017/p/3914149.html "同步与异步、阻塞与非阻塞")
* 通过4张图说明了同步阻塞、同步非阻塞、异步阻塞、异步非阻塞用户态和内核态的工作时序和内容。链接：[使用异步IO大大提高应用程序的性能之一](http://blog.chinaunix.net/uid-26000296-id-3754543.html "使用异步IO大大提高应用程序的性能之一")
* 试图通过通俗的例子说明差异，但是并没有讲清楚，就不要多看了，链接：[IO中同步、异步与阻塞、非阻塞的区别](http://blog.chinaunix.net/uid-26000296-id-3754118.html "IO中同步、异步与阻塞、非阻塞的区别")。不过，程序示例还可以参看：链接：[socket编程的同步、异步与阻塞、非阻塞示例详解之一](http://blog.chinaunix.net/uid-26000296-id-3755264.html "socket编程的同步、异步与阻塞、非阻塞示例详解之一")、链接：[socket编程的同步、异步与阻塞、非阻塞示例详解之二](http://blog.chinaunix.net/uid-26000296-id-3755268.html "socket编程的同步、异步与阻塞、非阻塞示例详解之二")。

<br/>
2015-3-22
