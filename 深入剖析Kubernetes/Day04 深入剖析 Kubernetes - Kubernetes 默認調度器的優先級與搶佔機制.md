# Day04 深入剖析 Kubernetes - Kubernetes 默認調度器的優先級與搶佔機制

## Kubernetes 默認調度器的優先級與搶佔機制
> Ref:
> - [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#how-to-use-priority-and-preemption)


**`優先級`（Priority）和`搶佔機制`（Preemption），解決的是 Pod 調度失敗時的問題。**

> 調度器的作用就是為 Pod 尋找一個合適的 Node 。

Kubernetes 限制，優先級是一個 32 bit 的整數，最大值不超過 1000000000（10 億，1 billion），並且值越大代表優先級越高。超過最大值的值，被保障系統 Pod 不會被用戶搶佔掉。

``` yaml
apiVersion: scheduling.k8s.io/v1beta1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for high priority service pods only."
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

**調度流程：**
- 待調度Pod被提交到 apiServer -> 
- 更新到 etcd ->
- 調度器 Watch etcd 感知到有需要調度的 Pod（Informer） ->
- 取出待調度 Pod 的訊息 -> 
- Predicates： 挑選出可以運行該 Pod 的所有 Node  -> 
- Priority：給所有 Node 打分 ->
- 將 Pod 綁定到得分最高的 Node 上 -> 
- 將 Pod 訊息更新回 Etcd -> 
- node 的 kubelet 感知到 etcd 中有自己 node 需要拉起的 Pod ->
- 取出該Pod 訊息，做基本的二次檢測（端口，資源等） ->
- 在 node 上拉起該 Pod 。

Predicates 階段很多過濾規則： 如 **volume 相關**，**node 相關**，**pod 相關**

Priorities階段會為 **Node 打分，Pod 調度到得分最高的 Node 上**，打分規則比如： 空余資源、實際物理剩餘、鏡像大小、Pod 親和性等。

Kuberentes 中可以為 Pod 設置優先級：

- **高優先級的 Pod：**
    - **在調度隊列中先出隊進行調度**
    - **調度失敗時，觸發搶佔，調度器為其搶佔低優先級 Pod 的資源**

Kuberentes 默認調度器有 2 個調度隊列：

- `activeQ`: 凡事在該隊列里的 Pod，都是**下一個調度週期需要調度的**
- `unschedulableQ`: **存放調度失敗的 Pod**，當裡面的 Pod 更新後就會重新回到 activeQ，進行重新調度

**默認調度器的搶佔流程：**
- 確定要發生搶佔 ->
- 調度器將所有節點訊息複製一份，開始模擬搶佔 ->  
- 檢查副本里的每一個節點，然後從該節點上逐個刪除低優先級 Pod，直到滿足搶佔者能運行 -> 
- 找到一個能運行搶佔者 Pod 的 Node -> 
- Pod 記錄下這個 Node 名字 (`status.nominatedNodeName`) 和被刪除 Pod 的列表 -> 
- 模擬搶佔結束 -> 
- 開始真正搶佔 -> 
- 刪除被搶佔者的 Pod，將搶佔者調度到 Node 上。


在為某一對 Pod 和 Node 執行 `Predicates` 算法的時候，如果待檢查的 Node 是一個即將被搶佔的節點，即：調度隊列里有 `nominatedNodeName` 字段值是該 Node 名字的 Pod 存在（潛在的搶佔者）。那麼，調度器就會對這個 Node ，將同樣的 `Predicates` 算法運行兩遍。

- 第一遍， 調度器會假設上述**潛在的搶佔者**已經運行在這個節點上，然後執行 `Predicates` 算法。（`InterPodAntiAffinity`）
- 第二遍， 調度器會正常執行 Predicates 算法，即：不考慮任何潛在的搶佔者。而**只有這兩遍 Predicates 算法都能通過時，這個 Pod 和 Node 才會被認為是可以綁定（bind）的**。

此文章為2月Day04學習筆記，內容來源於極客時間[《深入剖析Kuberentes》](https://time.geekbang.org/column/article/70519)