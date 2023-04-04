# 程序員練級攻略：分布式架構工程設計 (not yet)

左耳朵耗子在分布式的設計模式的總結：
- `彈力設計篇`，內容包括：認識故障和彈力設計、隔離設計、異步通訊設計、冪等性設計、服務的狀態、補償事務、重試設計、熔斷設計、限流設計、降級設計、彈力設計總結
- `管理設計篇`，內容包括：分布式鎖、配置中心、邊車模式、服務網格、網關模式、部署升級策略等
- `性能設計篇`，內容包括：緩存、異步處理、數據庫擴展、秒殺、邊緣計算等。


## 關於性能方面
- [Understand Latency]() ，這篇文章收集並整理了一些和系統響應時間相關的文章，可以讓你全面瞭解和 Latency 有關的系統架構和設計經驗方面的知識
- [Common Bottlenecks]() ，文中講述了 20 個常見的系統瓶頸
- [Performance is a Feature]() ，Coding Horror 上的一篇讓你關注性能的文章
- [CloudFlare: How we built rate limiting capable of scaling to millions of domains]()，講述了 CloudFlare 公司是怎樣實現他們的限流功能的。從最簡單的每客戶 IP 限流開始分析，進一步講到 anycast，在這種情況下 PoP 的分布式限流是怎樣實現的，並詳細解釋了具體的算法。

## 各公司的架構實踐
[High Scalability](http://highscalability.com) ，這個網站會定期分享一些大規模系統架構是怎樣構建的。
下面是迄今為止各個公司的架構說明：
- [YouTube Architecture](http://highscalability.com/youtube-architecture)
- [Scaling Pinterest](http://highscalability.com/blog/2013/4/15/scaling-pinterest-from-0-to-10s-of-billions-of-page-views-a.html)
- [Google Architecture](https://time.geekbang.org/column/article/11232)
- [Scaling Twitter](http://highscalability.com/scaling-twitter-making-twitter-10000-percent-faster)
- [The WhatsApp Architecture](http://highscalability.com/blog/2014/2/26/the-whatsapp-architecture-facebook-bought-for-19-billion.html)
- [Flickr Architecture](http://highscalability.com/flickr-architecture)
- [Amazon Architecture](http://highscalability.com/amazon-architecture)
- [Stack Overflow Architecture](http://highscalability.com/blog/2009/8/5/stack-overflow-architecture.html)
- [Pinterest Architecture](http://highscalability.com/blog/2012/5/21/pinterest-architecture-update-18-million-visitors-10x-growth.html)
- [Tumblr Architecture](http://highscalability.com/blog/2012/2/13/tumblr-architecture-15-billion-page-views-a-month-and-harder.html)
- [Instagram Architecture](http://highscalability.com/blog/2011/12/6/instagram-architecture-14-million-users-terabytes-of-photos.html)
- [TripAdvisor Architecture](http://highscalability.com/blog/2011/6/27/tripadvisor-architecture-40m-visitors-200m-dynamic-page-view.html)
- [Scaling Mailbox](http://highscalability.com/blog/2013/6/18/scaling-mailbox-from-0-to-one-million-users-in-6-weeks-and-1.html)
- [Salesforce Architecture](http://highscalability.com/blog/2013/9/23/salesforce-architecture-how-they-handle-13-billion-transacti.html)
- [ESPN Architecture](http://highscalability.com/blog/2013/11/4/espns-architecture-at-scale-operating-at-100000-duh-nuh-nuhs.html)
- [Uber Architecture](http://highscalability.com/blog/2015/9/14/how-uber-scales-their-real-time-market-platform.html)
- [Dropbox Design](https://youtu.be/PE4gwstWhmc)
- [Splunk Architecture](https://www.splunk.com/view/SP-CAAABF9)

## 小結

講述了設計原則、設計模式等方面的內容，尤其作者整理和推薦了國內外知名企業的設計思路和工程實踐，十分具有借鑒參考意義。

此文章為4月Day04學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/11232)