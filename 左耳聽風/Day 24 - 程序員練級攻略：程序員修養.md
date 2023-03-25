# Day 24 - 程序員練級攻略：程序員修養

## 程序員的修養

可以從[What are some of the most basic things every programmer should know?](https://www.quora.com/What-are-some-of-the-most-basic-things-every-programmer-should-know)了解什麼是程序員的修養？以下為其中一些語句：
- Bad architecture causes more problems than bad code.
- You will spend more time thinking than coding.
- The best programmers are always building things.
- There’s always a better way.
- Code reviews by your peers will make all of you better.
- Fewer features for better code is always the right answer in the end.
- If it’s not tested, it doesn’t work.
- Don’t reinvent the wheel, library code is there to help.
- Code that’s hard to understand is hard to maintain.
- Code that’s hard to maintain is next to useless.
- Always know how your business makes money, that determines who gets paid what.
- If you want to feel important as a software developer, work at a tech company.

還可以參考經典的[97 Things Every Programmer Should Know](https://97-things-every-x-should-know.gitbooks.io/97-things-every-programmer-should-know/content/en/index.html)，97 為其中 97 個不錯的編程建議。

## 英文能力

因為軟體開發技術的第一手消息，都來自於西方國家。所以如果想成為一個高手的話，那麼必須到訊息的源頭去。英文的世界真是有價值的訊息的集散地。
- 可以到官網上直接閱讀手冊，到 StackOverflow 上問問題，到 YouTube 上看很多演講和教學，到 GitHub 上參與社區，用 Google 查詢相關的知識，到國際名校上參加公開課等等

## 問問題的能力

提问的智慧（[How To Ask Questions The Smart Way](http://www.catb.org/~esr/faqs/smart-questions.html)）最早是由 Eric Steven Raymond 所撰寫的，詳細**描述了發問者事前應該做好什麼，而什麼又是不該做的**。作者**認為這樣能讓問題容易令人理解，而且發問者自己也能學到較多東西**。

另一個經典問題為 [XY Problem](https://xyproblem.info)。對於 `XY Problem` 的意思如下：

1. 有人想解決問題X
2. 他覺得Y可能是解決X問題的方法
3. 但是他不知道Y應該怎麼做
4. 於是他去問別人Y應該怎麼做？

簡而言之，**沒有去問怎麼解決問題X，而是去問解決方案Y應該怎麼去實現和操作**，結果**在錯誤方向上浪費了大家的時間**。

## 寫代碼的修養

可以參考：
- 代碼大全
- 重構：改善既有代碼的設計
    - Martin Fowler 的經典之作。這本書的意義**不僅僅在於改善既有代碼的設計，也指導了我們如何從零開始構建代碼的時候避免不良的代碼風格**。這是一本程序員必讀的書。
- 修改代碼的藝術
    - 不僅可以幫你掌握最頂尖的修改代碼技術，還可以大大提高你對代碼和軟件開發的領悟力。
- 代碼整潔之道
    - 代碼質量與其整潔度成正比。
- 程序員的職業素養
    - 作者以自己以及身邊的同事走過的彎路、犯過的錯誤為例，意在為後來人引路，助其職業生涯邁上更高台階。講解成為真正專業的程序員需要什麼樣的態度、原則，需要採取什麼樣的行動。

作為程序員，`Code Review` 和 `Unit Test` 為很重要的程序員修養。

Code Review 可以了解一下:
- [Code Review Best Practices](https://blog.palantir.com/code-review-best-practices-19e02780015f)
- [How Google Does Code Review](https://dzone.com/articles/how-google-does-code-review)
- [LinkedIn’s Tips for Highly Effective Code Review](https://thenewstack.io/linkedin-code-review/)

Unit Test 可以了解:
- [JUnit User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [You Still Don’t Know How to Do Unit Testing](https://stackify.com/unit-testing-basics-best-practices/)
- [Unit Testing Best Practices: JUnit Reference Guide](https://dzone.com/articles/unit-testing-best-practices)
- [JUnit Best Practices](http://www.kyleblaney.com/junit-best-practices/)


## 安全防範
在代碼中沒有最基本的安全漏洞問題，也是我們程序員必須確保的問題，尤其是對外暴露 Web 服務的軟件，其安全性就更為重要。對於 Web 安全問題，需要了解 [OWASP - Open Web Application Security Project](https://owasp.org)。

有一篇和 HTTP 相關的安全文章也是每個程序員必須要讀的: [Hardening Your HTTP Security Headers](https://www.keycdn.com/blog/http-security-headers/)

防禦性編程 ([Defensive Programming](https://en.wikipedia.org/wiki/Defensive_programming))，是為了保證對程序的不可預見的使用，不會造成程序功能上的損壞。

## 軟件工程和上線

表明你寫的軟件不是跑在自己的機器上的玩具，或是實驗室里的實驗品，而是交付給用戶使用的，甚至是用戶付費的軟件。需要遵守一些上線規範，比如，需要認真測試，並做上線前檢查，以及上線後監控。

關於測試，可以參考兩本書：
- 完美軟件：對軟件測試的各種幻想
- Google 軟件測試之道

下面兩個檢查清單，可以當做通用上線參考，再根據自己需求做調整：
- [Server Side checklist](https://github.com/mtdvio/going-to-production/blob/master/serverside-checklist.md)
- [Single Page App Checklist](https://github.com/mtdvio/going-to-production/blob/master/spa-checklist.md)

## 小結

有修養的程序員才可能成長為真正的工程師和架構師，而沒有修養的程序員只能淪為碼農。

需要留意學習方面的能力：英文能力、問問題的能力、寫代碼的修養、安全防範意識、軟件工程和上線規範等

此文章為3月Day24學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/8700)