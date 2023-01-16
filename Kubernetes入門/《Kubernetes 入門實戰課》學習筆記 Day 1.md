# 《Kubernetes 入門實戰課》學習筆記 Day 1

## Docker

學習docker使用及背後原理和技術：`cgroups`、`namespace`、`Union FS`

### cgroups

- 一種資源控制方案
- 對CPU、Memory、硬體I/O進行限制
- 可以用`Hierarchy`組織管理，sub-cgroup除了受到自身限制外，也受到parent-cgroup的資源限制

### namespace

- 一種 Linux Kernel提供資源隔離方案
- 為process提供不同的`namespace`進行區隔，互不干擾
- 每個`namespace`會包含以下這些

| Namespace Type | 隔離資源 |
| --- | --- |
| IPC | System V IPC 和 POSIX 消息對列 |
| Network | 網路設備、協議、端口等 |
| PID | 進程 process |
| Mount | 掛載點 |
| UTS | 主機名、域名 |
| USR | 用戶、用戶組 |
- 課後練習

```bash
### Namespace
# 在new network namespace 執行 sleep指令
unshare -fn sleep 60
# 查看進程
ps -ef | grep sleep
root 32882 4935 0 10:00 pts/0 00:00:00 unshare -fn sleep 60
root 32883 32882 0 10:00 pts/0 00:00:00 sleep 60
# 查看 Network namespace
lsns -t net
4026532508 net 2 32882 root unassigned unshare
# 進入進程，查看ip
nsenter -t 32882 -n ip a
1: lo <loopback> mtu 65536 ...
		link/loopback 00:00:00 ...

### Cgroups
# 新建一個test dir：cpudemo
cd /sys/fs/cgroup/cpu/cpudemo
# 找到busyloop的PID，加入到cgroup.proc
ps -ef | grep busyloop
echo <pid> > cgourp.proc
# 或是直接透過下面指令加入
ps -ef | grep busyloop | grep -v grep | awk '{print $2}' > cgroup.proc
#  cpu.cfs_period_us 用來配置CPU時間週期長度
#  cpu.cfs_quota_us 當前cgroup在cpu.cfs_period_us中最多能使用的cpu時間數
echo 10000 > cpu.cfs_quota_us
# 執行 top 查看CPU使用情況 是否變為 10%
```

- union FS (file system)
    - 將不同目錄掛載到同一個虛擬文件系統下
    - 另一個更常用的是將一個`readonly`的branch和一個`writeable`的branch聯合一起

### docker

- 啟動將`rootfs`以`readonly`方式讀取並檢查，利用`union mount`將一個`readwrite`文件掛載在`readonly`的`rootfs`上
- 一組`readonly`和`writeable`結構為一個`container`運作方式，為一個`FS`層。並可以將下層的`FS`設為`readonly`向上疊加
- 鏡像具有共享特性，所以對容器可寫層操作需要依靠存儲驅動提供的機制來支持，提供對存儲和資源的使用率
    - 寫時複製 COW
        - copy-on-write
        - 從鏡像文件系統複製到容器的可寫層文件系統進行修改，跟原本的文件，相互獨立
    - 用時分配
        - 被創建後才分配空間
- 課後練習

```bash
# OverlayFS practice
mkdir upper lower merged work
echo "from lower" > lower/in_lower.txt
echo "from upper" > lower/in_upper.txt
echo "from lower" > lower/in_both.txt
echo "from upper" > lower/in_both.txt

sudo mount -t overlay overlay -o lowerdir=`pwd`/lower, upperdir=`pwd`/upper, workdir=`pwd`/work `pwd`/merged
cat merged/in_both.txt
```

- 課後練習
    - 因OS結構為 `linux/arm64`，課後環境都需要而外建立無法用正常的`amd64`鏡像
        - 此在縮小鏡像體積時候，遇到以下錯誤
            - solved： 在`dockerfile`中加入`cgo=0`的環境變數
                - 因為會有不同平台會有動態連結的問題
    - 鏡像可以透過多端構建，只留build好的部分，排除過程產物，減少鏡像體積
      
        ```docker
        # 前者路徑為建置好的檔案路徑
        COPY --from=builder /build/demo/http-server /
        ```
        
    - `golang:scratch`和`golang:alpine`鏡像
        - 前者雖然體積小，但是沒包含基本除錯工具
    
    ```markdown
    # Resize Dockerfile
    ```dockerfile
    FROM golang:1.18-alpine AS builder
    
    ENV CGO_ENABLED=0 
    #   GO111MODULE=off  \
    #	GOOS=linux    \
    #	GOARCH=amd64
    
    WORKDIR /build
    COPY . .
    RUN echo "Install dependent modules" && \
        go mod download && go mod verify && \
        cd demo/ && \
        go build -o http-server .
    
    FROM busybox
    COPY --from=builder /build/demo/http-server /
    EXPOSE 8080
    CMD ["/http-server"]
    #ENTRYPOINT ["/http-server"]
    ```
    ```
## Kubernetes

- 了解前身Google Borg的來歷
- 基於容器的應用部署、維護、滾動升級

### 聲明式API，核心對象

- `Node`
    - 節點的抽象，描述計算節點的狀況
    - `Node`是`Pod`真正運行的主機
- `Namespace`
    - 資源隔離的基本單位
    - 一組資源和對象的抽象集合
- `Pod`
    - 用來描述應用實例，kubernets最核心對象，也是調度的基本單位
    - 同一個`Pod`中的不同容器課共享資源
        - 共享`Network Namespace`
        - 可通過掛載`存儲卷`共享存儲
            - 存儲卷：從外部存儲掛載到`Pod`內部使用
            - 分為兩部分：`Volume`和`VolumeMounts`
                - Volume：定義Pod可以使用的存儲來源
                - VolumeMounts：定義如何掛載到容器內部
        - 共享`Security Context`
    - 單機限制`110`個Pod
- `Service`：將應用發布成服務，本質是負載均衡和域名服務的聲明
- 每個API對象都有四大屬性

### TypeMeta

- 通過此引GKV（`Group`, `Kind`, `Version`）模型定義一個對象類型
    - `Group`：將對象依據其功能分組
    - `Kind`：定義對象的基本類型
        - e.g. Node、Pod、Deployment等
    - `Version`：每季度會推出Kubernetes版本
        - e.g. v1alpha1、v1alpha2、v1（生產版本）

### MetaData

- 兩個重要屬性：`Namespace`和`Name`，分別定義對象的namespace歸屬，這兩屬性唯一定義了對象實例
    - Label
        - `KV-pairs`，kubernetes api支持以`label`作為過濾條件
        - label selector支持以下方式
            - 等式，如`app=nginx`或是`env≠production`
            - 集合，如 `env in (production, qa)`
    - Annotation
        - `KV-paris`，此為屬性擴展，更多面向開發及管理人員，所以需要像其他屬性合理歸類
        - 用來記錄一些附加訊息，如`deployment`使用`annotation`來記錄`rolling update狀態`
    - Finalizer
        - 本質為一個資源鎖
        - kubernetes在接受對象的刪除請求時，會檢查`Finalizer`是否為空，不為空只做邏輯刪除，即只會更新對象中的`metadata.deletionTimestamp`字段
    - ResourceVersion
        - 類似一個樂觀鎖
        - Kubernetes對象被客戶端讀取後`ResouceVersion`訊息也同時被讀取。此機制確保了分布式系統中任意多線程能夠無鎖併發訪問對象

### Spec & Status

- Spec：用戶期望方式，由用戶自行定義
    - 健康檢查
        - 探針類型
            - LivenessProbe
                - 檢查應用是否健康，若否則刪除並重新創建容器
            - ReadinessProbe
                - 檢查應用是否就緒且為正常服務狀態，若否則不會接受來時`Kubernets Services`的流量
            - StartupProbe
                - 檢查應用是否啟動成功，如果在`failureThreshold*periodSeconds`週期內為就緒，則應用會被重啟
        - 探活方式
            - Exec
            - Tcp socket
            - Http
- Status：對象實際狀態，由對應`Controller`收集狀態並更新
- 跟通用屬性不同，`Spec`和`Status`是每個對象獨有的
