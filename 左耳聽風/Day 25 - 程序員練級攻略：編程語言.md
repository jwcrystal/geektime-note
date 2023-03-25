# Day 25 - 程序員練級攻略：編程語言

為了進入專業的編程領域，需要認真學習以下三方面的知識。

## 編程語言

**需要了解 C、C++ 和 Java 這三個工業級的編程語言**。為什麼說它們是工業級的呢？主要是，C 和 C++ 語言規範都由 ISO 標準化過，而且都有工業界廠商組成的標準化委員會來制定工業標準；次要原因是，它們已經在業界應用於許多重要的生產環境中。

此外，也推薦學習 Go 語言。一方面，Go 語言現在很受關注，它是取代 C 和 C++ 的另一門有潛力的語言。C 語言太原始了，C++ 太複雜了，Java 太高級了，所以 Go 語言就在這個夾縫中出現了。這門語言已經 10 多年了，其已成為雲計算領域事實上的標準語言，尤其是在 **Docker/Kubernetes** 等項目中。Go 語言社區正在不斷地從 Java 社區移植各種 Java 的輪子過來，Go 社區現在也越来越龐大了。

## 理論知識

需要學習像**算法、數據結構、網絡模型、計算機原理**等計算機科學專業需要學習的知識。
為什麼要學好這些理論上的知識呢？
- 其一，這些理論知識可以說是計算機科學這門學科最精華的知識了。說得大一點，這些是人類智慧的精華。只要想成為高手，這些東西是你必需要掌握和學習的
- 其二，當在解決一些很複雜或是很難的問題時，這些基礎理論知識反而會幫到很多
- 其三，這些理論知識的思維方式可以讓你有觸類旁通，一通百通的感覺。雖然知識比較難啃，但啃過以後，你將獲益終生。

## 系統知識

系統知識是理論知識的工程實踐。這些知識是你能不能把理論應用到實際項目當中，能不能搞定實際問題的重要知識。
- 如像 **Unix/Linux、TCP/IP、C10K** 挑戰等這樣專業的系統知識
- 如在編程的時候，如何和系統進行交互或是獲取操作系統的資源，如何進行通訊，當系統出了性能問題，當系統出了故障等，你有大量需要落地的事需要處理和解決。這個時候，這些系統知識就會變得尤為關鍵和重要

## Java 

參考資源：
- Java 核心技術：卷 1 基礎知識
- Spring Boot 實戰
- **Effective Java**
- [**Google Guava 庫**](https://github.com/google/guava)
- **Java 併發編程實戰**
- Java 性能權威指南
- 深入理解 Java 虛擬機
- **Java 編程思想**
- **设计模式 or Head First 設計模式**
    - **Program to an interface, not an implementation**
        - 使用者不需要知道數據類型、結構、算法的細節
        - 使用者不需要知道實現細節，只需要知道提供的接口
        - 利於抽象、封裝，動態綁定，多態。符合面向對象的特質和理念
    - **Favor object composition over class inheritance**
        - 繼承需要給子類暴露一些父類的設計和實現細節
        - 父類實現的改變會造成子類也需要改變
        - 我們以為繼承主要是為了代碼重用，但實際上在子類中需要重新實現很多父類的方法
        - 繼承更多的應該是為了多態

## C/C++

參考資源：
- [C 程序設計語言](https://book.douban.com/subject/1139336/)
- [C 語言程序設計現代方法](https://book.douban.com/subject/2280547/)
- [C 陷阱與缺陷](https://book.douban.com/subject/2778632/)
- [C++ Primer 中文版](https://book.douban.com/subject/25708312/)
- [Effective C++](https://book.douban.com/subject/5387403/)
- [More Effective C++](https://book.douban.com/subject/5908727/)
- [深度探索 C++ 對象模型](https://book.douban.com/subject/10427315/)
- [C++ FAQ](https://www.stroustrup.com/bsfaqcn.html)

## Go

參考資源：
- **[Go by Example](http://books.studygolang.com/gobyexample/)**
- [Go 101](https://go101.org/article/101.html)
- **[Go 語言聖經](https://books.studygolang.com/gopl-zh/)**
- **[Effective Go](https://go.dev/doc/effective_go)**
- **[Go Concurrency Patterns](https://go.dev/talks/2012/concurrency.slide#1)** / [演講影片](https://youtu.be/f6kdp27TYZs)
- **[Advanced Go Concurrency Patterns](https://go.dev/talks/2013/advconc.slide#1)** / [演講影片](https://youtu.be/QDDwwePbDtw)
- [Awesome-Go](https://awesome-go.com)


## 小結

一個合格的程序員應該掌握幾門語言。一方面，這會讓你對不同的語言進行比較，讓你有更多的思考。另一方面，這也是一種學習能力的培養，會讓你對於未來的新技術學習得更快。

此文章為3月Day25學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/8701)