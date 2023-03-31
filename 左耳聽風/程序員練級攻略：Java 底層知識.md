# 程序員練級攻略：Java 底層知識


## Java 字節碼

Java 最黑科技的玩法就是字節碼編程，也就是**動態修改或是動態生成 Java 字節碼**。

一般來說，我們不使用 JVMTI 操作字節碼，而是用一些更好用的庫。這裡有三個庫可以幫你比較容易地做這個事：
- `asmtools` - 用於生產環境的 Java .class 文件開發工具
- `Byte Buddy` - **代碼生成庫：運行時創建 Class 文件而不需要編譯器幫助**
- `Jitescript` - 和 BiteScript 類似的字節碼生成庫。

使用字節碼編程可以玩出很多高級玩法，最高級的還是在 **Java 程序運行時進行字節碼修改和代碼注入**。這個方式使得 Java 這門靜態語言在運行時可以進行各種動態的代碼修改，而且可以進行無侵入的編程。

- 比如，我們不需要在代碼中埋點做統計或監控，可以使用這種技術把我們的監控代碼直接以字節碼的方式注入到別人的代碼中，從而實現對實際程序運行情況進行統計和監控
> 需要使用 Java Agent 的技術，其主要方法是實現一個叫 premain() 的方法 。
> 
> 在 JVM 啓動時，使用這樣的命令行來引入你的 jar 文件：`java -javaagent:yourAwesomeAgent.jar -jar App.jar`。更為詳細的文章你可以參看：[Java Code Geeks: Java Agents](https://www.javacodegeeks.com/2015/09/java-agents.html)，你還可以看一下這個示例項目：[jvm-monitoring-agent](https://github.com/toptal/jvm-monitoring-agent) 或是 [EntryPointKR/Agent.java](https://gist.github.com/EntryPointKR/152f089f6f3884047abcd19d39297c9e)。如果想用 ByteBuddy 來玩，可以看看這篇文章[通過使用 Byte Buddy，便捷地創建 Java Agent](https://www.infoq.com/cn/articles/Easily-Create-Java-Agents-with-ByteBuddy)。如果想學習如何用 Java Agent 做監控，可以看一下這個項目 [Stage Monitor](https://www.stagemonitor.org)。

## JVM

系統了解 JVM 規範的最佳文檔：
- [The Java Virtual Machine Specification Java SE 8 Edition](https://docs.oracle.com/javase/specs/jvms/se8/jvms8.pdf)
- [The Java Language Specification Java SE 11 Edition](https://docs.oracle.com/javase/specs/jls/se11/html/index.html)
- [The Java® Virtual Machine Specification.《Java 虚拟机规范（第11版）》中文翻译及示例](https://github.com/waylau/java-virtual-machine-specification)

Java 內存模型：
- [官方文章 JSR 133](https://www.jcp.org/en/jsr/detail?id=133)
- [The Java Memory Model](https://www.cs.umd.edu/~pugh/java/memoryModel/)
- [JVM Anatomy Park](https://shipilev.net/jvm/anatomy-quarks/)

Java Gabage Collection:
- [The Garbage Collection Handbook](https://book.douban.com/subject/6809987/)
- [Garbage Collection Tuning Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/)
- [Quick Tips for Fast Code on the JVM](https://gist.github.com/djspiewak/464c11307cabc80171c90397d4ec34ef)

## 小結

學習內容：
- **Java 字節碼**相當於匯編，即可以動態修改或是動態生成 Java 字節碼
- 使用 **Java Agent** 技術完成 Java 程序運行時進行字節碼修改和代碼注入
- 了解到 **JVM**、**Java 的內存模型**、**垃圾回收機制**，尤其給出了如何調優垃圾回收方面的資料

此文章為3月Day31學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/10216)