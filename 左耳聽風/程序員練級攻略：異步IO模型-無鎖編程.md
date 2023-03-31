# 程序員練級攻略：異步IO模型-無鎖編程
> https://time.geekbang.org/column/article/9851

## 異步 I/O 模型

5 種異步 I/O 模型：
- 阻塞 I/O
- 非阻塞 I/O
- I/O 多路復用（select 和 poll）
- 信號驅動的 I/O（SIGIO）
- 異步 I/O（POSIX 的 aio_functions）

異步 I/O 模型的發展技術是： select -> poll -> epoll -> aio -> libevent -> libuv

- [Thousands of Threads and Blocking I/O: The Old Way to Write Java Servers Is New Again (and Way Better)](https://www.slideshare.net/e456/tyma-paulmultithreaded1)

## Lock-Free 
- [Dr.Dobb’s: Lock-Free Data Structures](https://www.drdobbs.com/lock-free-data-structures/184401865)
- [Andrei Alexandrescu: Lock-Free Data Structures](https://erdani.com/publications/cuj-2004-10.pdf)
- [Is Parallel Programming Hard, And, If So, What Can You Do About It?](https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html)

異步 I/O 模型是所有程序員都必需要學習的一門技術或是編程方法，這其中的設計模式或是解決方法可以借鑒到分布式架構上來。

如果想開發出一個高性能的程序，有必要學習 **Lock-Free** 的編程方式.

此文章為3月Day30學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/9759)