# Day 28 - 程序員練級攻略：軟件設計

學習軟件設計的方法、理念、範式和模式，是讓你從一個程序員通向工程師的必備技能。軟件開發實現功能，做出來並不難，但是要做漂亮，做優雅，就不是那麼容易。

**要學好這些軟件開發和設計的方法，真的需要磨練和苦行，反復咀嚼，反復推敲，在實踐和理論中螺旋式地學習，才能真正掌握。**

## 編程範式

參考資源：

- 作者左耳朵耗子的[編程范式遊記](https://time.geekbang.org/column/article/301)。
- [Six programming paradigms that will change how you think about coding](https://www.ybrikman.com/writing/2014/04/09/six-programming-paradigms-that-will/)

## 設計原則

SOLID 原則：
- SRP 單一職責原則：單一模塊只對一件事情（一類行為）負責
- OCP 開閉原則：針對擴展開放，修改關閉
- LSP 里式替換原則：子類型可以替換父類型
- ISP 接口隔離原則：只依賴需要的接口
- DIP 依賴倒置原則：高層組件不會互相依賴

DRY 原則、KISS 原則、YAGNI 原則、LOD 法則：
- DRY： Don't repeat yourself，不做重複的事情
- KISS： Keep it simple and stupid，盡量簡單代碼，讓人容易理解
- YANGI： You ain't gonna need it，只加入需要的功能
- LOD： Law of demeter，狄米特原則 (最少知識原則)，對其他對象保持最少的了解

CCP、CRP、CoC、SoC、DbC、ADP 原則：
- CCP：Common Closure Principle，一個包中所有的類應該對同一種類型的變化關閉
- CRP：Common Reuse Principle，包的所有類被一起重用
- CoC：Convention over Configuration，**將一些公認的配置方式和信息作為內部預設的規則來使用**
- SoC：Separation of Concerns，就是在軟件開發中，通過各種手段，將問題的各個關注點分開
- DbC：Design by Contract，對軟件系統中的**元素之間相互合作以及責任與義務**的比喻
- ADP：Acyclic Dependencies Principle，包（或服務）之間的依賴結構必須是一個直接的無環圖形
    - 有兩種方法可以打破這種循環依賴關係：
        1. 創建新的包
        2. 使用 DIP（依賴倒置原則）和 ISP（接口分隔原則）設計原則
    - 無環依賴原則（ADP）為我們解決包之間的關係耦合問題。在設計模塊時，不能有循環依賴

## 軟件設計參考

- [領域驅動設計](https://book.douban.com/subject/26819666/)
- [UNIX 編程藝術]()
- [Clean Architecture](https://book.douban.com/subject/26915970/)
- [The Twelve-Factor App](https://12factor.net)
- [Avoid Over Engineering ](https://medium.com/@rdsubhas/10-modern-software-engineering-mistakes-bc67fbef4fc8)
- [Instagram Engineering’s 3 rules to a scalable cloud application architecture](https://datastax.medium.com/instagram-engineerings-3-rules-to-a-scalable-cloud-application-architecture-c44afed31406)
- [How To Design A Good API and Why it Matters - Joshua Bloch](https://www.infoq.com/presentations/effective-api-design)
- [The Problem With Logging](https://blog.codinghorror.com/the-problem-with-logging/)
- [Concurrent Programming for Scalable Web Architectures](http://berb.github.io/diploma-thesis/community/index.html)

## 小結

學習編程範式，能夠幫助明白編程的本質和各種語言的編程方式。

想要成為軟件工程師、設計師或架構師，可以透過學習上文內容，能夠提高軟件設計的知識，進而實現自己的目標。

此文章為3月Day28學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/9369)