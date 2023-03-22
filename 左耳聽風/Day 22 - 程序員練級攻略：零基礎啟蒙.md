# Day 22 - 程序員練級攻略：零基礎啟蒙

從零基礎開始的人來說，最重要的是能夠對編程有興趣，而**要對編程有興趣，就要有成就感**。

## 編程入門

使用 `Python` 和 `JavaScript` 作為入門語言。Python 語法簡單，有大量的庫和語法糖，是零基礎的人學習編程的最佳選擇。而 JavaScript 則是前端的語言，頁面直接呈現出來，容易更有編程的成就感，所以也成了一門要學習的語言。

### Python

零基礎入門建議書籍。主要是**通過書中的示例來強化對編程的學習**。第一本偏文本處理，包括處理 Word、Excel 和 PDF，第二本中有一些 Web 項目和代碼部署方面的內容。

如果時間有限，可以先選擇第二本閱讀。

- [Python 編程快速上手](https://book.douban.com/subject/26836700/)
- [Python 編程：從入門到實踐](https://book.douban.com/subject/26829016/)

### JavaScript

- [MDN JavaScript 教程](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript)，最具權威和官方建議的學習地方
- [W3School JavaScript 教程](https://www.w3school.com.cn/js/index.asp)，偏 Web 方面的編程
- [JavaScript 全栈教程（廖雪峰）](https://www.liaoxuefeng.com/wiki/1022910821149312)

## 操作系統入門 Linux

學習編程還需要會玩 Linux。
- [Linux 教程](https://www.w3cschool.cn/linux/)

## 編程工具 Visual Studio Code

目前主流且輕量的編程工具 Visual Studio Code，插件豐富，用它開發 Python、JavaScript、Java、Go、C/C++...等等，都很適合。
- [Visual Studio Code 教程](https://jeasonstudio.gitbooks.io/vscode-cn-doc/content/)

## Web 編程入門

- **前端基礎**。要系統地學習一下前端的知識，也就是 CSS、HTML 和 JavaScript 這三個東西。這裡還是給出 MDN 的相關的技術文檔頁面 [CSS 文檔](https://developer.mozilla.org/zh-CN/docs/Web/CSS) 和 [HTML 文檔](https://developer.mozilla.org/zh-CN/docs/Web/HTML)。**文檔很大，並不是要學習所有的東西，而是了解 CSS 和 HTML 是怎麼相互作用來展示數據的，然後，不用記憶文檔中的內容，這兩個文檔是用來查找知識的**。 另外，你可以簡單地學習使用 JavaScript 操縱 HTML。理解 DOM 和動態網頁（可以參看 [W3Schools 的 JavaScript HTML DOM 的教程](https://www.w3schools.com/js/js_htmldom.asp)）

- **後端基礎**。可以直接採用前面提到的 **Python** 或者 **Node.js**（一樣使用 JavaScript），這兩個技術在前面提到的廖雪峰的那個教程里提到過。或是直接學習目前主流的後端語言：**Java**、**C#**、**Go**、**PHP**等等。

下面是一些學習要點：
- 學習 HTML 基本語法
- 學習 CSS 如何選中 HTML 元素並應用一些基本樣式
- 學會用 Firefox + Firebug 或 Chrome 查看你覺得很炫的網頁結構，並動態修改
- 在一台 Linux 機器上配置 LEMP - Ubuntu/Nginx/PHP/MySQL 這個環境
- 學習後端開發，讓後台系統和前台 HTML 進行數據交互，對服務器響應瀏覽器請求形成初步認識，並實現一個表單提交和反顯的功能
- 學習後端如何連接本地或遠端數據庫 MySQL

## 實踐項目

可以練習寫一個簡單的 Blog 系統，支持簡單呈現功能即可：
- 用戶登錄和註冊（不需密碼找回）
- 用戶發貼（不需要支持富文本，只需要支持純文本）
- 用戶評論（不需要支持富文本，只需要支持純文本）

這些功能，就包含了前端和後端，到數據庫的開發及使用。

這裡有幾個技術點需要關注一下：
- 用戶登錄時的密碼不應該保存為明文，應該用 **MD5+Salt** 來保存
- 用戶登錄後，對於用戶自己的貼子可以有**重新編輯**或**刪除**的功能，但是無權編輯或刪除其它用戶的貼子
- 數據庫的設計，需要建立三張表：**用戶表**、**文章表**和**評論表**，需要學習一下它們之間是怎麼關聯的
    - 可以參考 PHP 的 blog 學習如何建表，[參考鏈接](https://code.tutsplus.com/tutorials/how-to-create-a-phpmysql-powered-forum-from-scratch--net-10188)。

進階版，可以接續加入以下功能：
- 圖片驗證碼
- 上傳圖片
- 阻止用戶在發文章或評論時輸入帶 HTML 或 JavaScript 的內容
- 防範 SQL 注入（可以參考[微軟官方文檔](https://learn.microsoft.com/zh-cn/previous-versions/sql/sql-server-2008-r2/ms161953(v=sql.105)?redirectedfrom=MSDN)）

## 小結

將會學習如何入門編程。從簡單入手的程式語言，到使用工具，再到相關學習範疇，最後透過實戰項目，幫助理解及鞏固所學的知識。

此文章為3月Day22學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/8136)