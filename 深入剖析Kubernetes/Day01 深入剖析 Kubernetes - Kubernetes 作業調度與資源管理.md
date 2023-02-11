# Day01 深入剖析 Kubernetes - Kubernetes 作業調度與資源管理

## Kubernetes 資源模型與資源管理
> Ref:
> - [Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/)
> - [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)


Kubernetes 的資源管理與調度部分的基礎，為資源模型。

- 在 Kubernetes 中，Pod 是最小原子調度單位，也**說明所有跟調度和資源管理相關的屬性都是屬於 Pod 對象的字段**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**CPU 資源為 `可壓縮資源（Compressible Resources）`，特點為當可壓縮資源不足時，Pod 只會飢餓，但不會退出。**

**Memory 內存資源為 `不可壓縮資源（Incompressible Resources）`，當該資源不足時，Pod 就會因為 OOM（Out-of-Memory）被內核回收掉。**

Pod 可由多個 Container 組成，因此，Pod 整體資源配置，為這些 Container 的配置的總和

### limits & requests

需要留意 MiB（mebibyte）和 MB（megabyte）的區別
> 1Mi = 1024 * 1024 , 1M  = 1000 * 1000

- `requests`：**kube-scheduler** 只會按照 `requests` 的值進行計算
- `limits`：**kubelet** 則會按照 `limits` 的值來進行設置 `Cgroups` 限制

### Qos 模型

主要應用場景，是**當宿主機資源緊張的時候，kubelet 對 Pod 進行 `Eviction`（即資源回收)** 時需要用到。

- `BestEffort`: 沒有設置，即沒有限制

- `Burstable`: request < limit

    - 不滿足 Guaranteed 條件， 但至少有一個 Container 設置了 requests
    
- `Guaranteed`: request = limit

    - 只有設置 limit，kubernetes 默認為 Guaranteed

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```

### Eviction 

計算 `Eviction` 閾值的數據來源，主要依賴於從 `Cgroups` 讀取到的值，以及使用 `cAdvisor` 監控到的數據。

- Eviction 默認值

```
memory.available < 100Mi
nodefs.available < 10%
nodefs.inodesFree < 5%
imagefs.available < 15%
```

- 可以透過指令 `kubelet` 進行配置

```shell
$ kubelet --eviction-hard=imagefs.available<10%,memory.available<500Mi,nodefs.available<5%,nodefs.inodesFree<5% --eviction-soft=imagefs.available<30%,nodefs.available<10% --eviction-soft-grace-period=imagefs.available=2m,nodefs.available=2m --eviction-max-pod-grace-period=600
```

- Eviction 分為 2 種：

    - `Soft Eviction`: 為Eviction 設置一段優雅時間
        
        - e.g. 如 `imagefs.available=2m`，表示當 imagefs 不足的閾值達到 2 分鐘之後，kubelet 才會開始 Eviction 的過程。
    
    - `Hard Eviction`: 超過閾值直接進行回收

### CPUSET

是生產環境里部署在線應用類型的 Pod 時，非常常用的一種方式。

在 kubernetes 中，只要滿足下面條件即為 cpuset 模式：

- Pod Qos 為 Guaranteed
- Pod CPU 的 requests 和 limits 設置為同一個相等的整數值

下面例子為 **Pod 獨佔 2 個 kubelct 分配的 CPU**
```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "2"
      requests:
        memory: "200Mi"
        cpu: "2"
```

## 小結

最佳實踐：

- **DaemonSet 的 Pod 都設置為 Guaranteed， 避免進入 **回收->創建->回收->創建** 的死循環**。

為什麼宿主機進入 `MemoryPressure` 或者 `DiskPressure` 狀態後，新的 Pod 就不會被調度到這台宿主機上呢？

- 因為宿主機被打了`污點（tiant）`標記

此文章為2月Day01學習筆記，內容來源於極客時間[《深入剖析Kuberentes》](https://time.geekbang.org/column/article/69678)