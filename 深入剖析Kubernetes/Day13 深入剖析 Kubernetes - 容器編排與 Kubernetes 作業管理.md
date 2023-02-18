# Day13 深入剖析 Kubernetes - 容器編排與 Kubernetes 作業管理

## Operator 工作原理

Etcd Operator Workflow

- 與常規的 Operator 在業務邏輯上有些許不同

![](media/16766973483938/16767120889498.jpg)

`Etcd Operator` 的特殊之處在於，它為每一個 `EtcdCluster 對象`，都啓動了一個控制循環，**併發地響應這些對象的變化**。顯然，這種做法不僅可以簡化 `Etcd Operator` 的代碼實現，還有助於提高它的響應速度。

此文章為2月Day13學習筆記，內容來源於極客時間[《深入剖析Kuberentes》](https://time.geekbang.org/column/article/42493)

《Linux0.11源碼趣讀》第二季重磅上線