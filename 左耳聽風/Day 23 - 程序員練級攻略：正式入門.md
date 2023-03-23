# Day 23 - 程序員練級攻略：正式入門

無論做什麼事，都會面對各式各樣的困難，這對每個人來說都是一樣的，而只**有興趣、熱情和成就感**才能讓自己不畏懼這些困難。

## 編程技能

系統學習編程前，可以參閱[The Key To Accelerating Your Coding Skills](http://blog.thefirehoseproject.com/posts/learn-to-code-and-be-self-reliant/)，文章討論如何快速提高編程能力。

入門內容分為：
- 編程技巧
    - 可以參考[代碼大全](http://blog.thefirehoseproject.com/posts/learn-to-code-and-be-self-reliant/)。雖然書籍有年紀了，但是**每個階段閱讀都可以獲得不同的收穫，甚至會有更深的思考角度**。
- 編程語言
    - 目前主流最全面的為 Java。建議從[Head First Java](https://book.douban.com/subject/2000732/) 著手，接著可以看著名框架[Sprint Boot](https://spring.io/quickstart)
    - C# 可以閱讀[官網 C# 文件](https://learn.microsoft.com/zh-tw/dotnet/csharp/)
    - Go 可以閱讀[Go語言聖經（中文版）](https://books.studygolang.com/gopl-zh/)
- 操作系統
    - 可以閱讀[鳥哥的 Linux 私房菜](https://linux.vbird.org)。可以對操作系統及 Linux 有非常全面的了解
- 網路協議
    - 可以閱讀[MDN HTTP 文檔](https://developer.mozilla.org/zh-CN/docs/Web/HTTP)。需要了解 HTTP 協議的幾個關鍵：
        - HTTP Header
        - HTTP 請求
        - HTTP 返回碼
        - HTTP Cookie、Cache、Session，及鏈接管理等等
- 數據庫設計
    - 需要了解如何操作數據庫和 SQL 的用法，可以學習開源的 MySQL，可以閱讀[MySQL 必知必會](https://book.douban.com/subject/3354490/)
- 前端方面
    - 上一份筆記提到的，HTML、JavaScript、CSS。還可以閱讀和 CSS 相關的 [Bootstrap](https://getbootstrap.com)
- 字符編碼
    - 處理中文遇到亂碼，就需要了解 ASCII 和 Unicode 的字符編碼，可以閱讀[關於字符編碼，你所需要知道的（ASCII,Unicode,Utf-8,GB2312…）](http://www.imkevinyang.com/2010/06/关于字符编码，你所需要知道的.html)

## 編程工具

- IDE
    - 主流為 Intellij IDEA，或是使用 Visual Studio Code
- 版本管理工具
    - 學習如何使用 Git。建議閱讀[Pro Git 2nd](https://git-scm.com/book/zh/v2/)，或是[猴子都能懂的 Git 入門](https://backlog.com/git-tutorial/cn/)，以及學習使用 GitHub 或是 Gitlab。
- 調試前端程序
    - 學習如何使用瀏覽器調試前端程序，可以參考[超完整的 Chrome 瀏覽器客戶端調試大全](http://www.igeekbar.com/igeekbar/post/156.htm)
- 數據庫設計工具
    - 可以學習[MySQL WorkBench](https://dev.mysql.com/doc/refman/5.7/en/) 

## 實戰項目

這回我們需要設計一個投票系統的項目。
業務上的需求如下：
- 用戶只有在登錄後，才可以生成投票表單
- 投票項可以單選，可以多選
- 其它用戶投票後顯示當前投票結果（但是不能刷票）
- 投票有相應的時間，頁面上需要出現倒計時
- 投票結果需要用不同顏色不同長度的橫條，並顯示百分比和人數

技術上的需求如下：
- 使用 Java Spring Boot 來實現了，然後，後端不返回任何的 HTML，只返回 JSON 數據給前端
- 由前端的 JQuery 來處理並操作相關的 HTML 動態生成在前端展示的頁面
- 前端的頁面還要是響應式的，也就是可以在手機端和電腦端有不同的呈現。這個可以用 Bootstrap 來完成。

進階版，還可以挑戰以下這些功能：
- 在微信中，通過微信授權後記錄用戶信息，以防止刷票
- 可以不用刷頁面，就可以動態地看到投票結果的變化
- Google 一些畫圖表的 JavaScript 庫，然後把圖表畫得漂亮一些

## 小結

如果能獨立實現上面提到的實戰項目，可以算是入門了，也算是一名全端工程師了。

那麼下一步**建議是選擇一個方向開始深入**。因為你並不知道你未來會有多大的可能性，也不知道你會成為什麼樣的人，如果不繼續深入研究且進步，那跟一般的代碼搬運工有什麼區別？

此文章為3月Day23學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/8216)