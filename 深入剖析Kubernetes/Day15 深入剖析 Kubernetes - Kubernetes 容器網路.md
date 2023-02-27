# Day15 深入剖析 Kubernetes - Kubernetes 容器網路

## 深入解析容器跨主機網路

在 Docker 的默認配置下，不同宿主機上的容器通過 IP 地址進行互相訪問是無法做到的。
> 因為網橋 docker0 無法直接溝通。為了解決**跨主通訊**，才有現在各種的容器網路方案。

Flannel 項目是 CoreOS 公司主推的容器網路方案。事實上，Flannel 項目本身只是一個框架，真正為我們提供容器網路功能的，是 Flannel 的後端實現。
目前，Flannel 支持三種後端實現，分別是：
- VXLAN；
- host-gw；
- UDP

### Flannel - UDP

UDP 模式，是 Flannel 項目最早支持的一種方式，卻也是性能最差的一種方式。

以例子說明 `UDP` 模式，假設兩台宿主機，container-1 要連到 container-2：

- 宿主機 Node 1 上有一個容器 container-1，它的 IP 地址是 `100.96.1.2`，對應的 docker0 網橋的地址是：`100.96.1.1/24`
- 宿主機 Node 2 上有一個容器 container-2，它的 IP 地址是 `100.96.2.3`，對應的 docker0 網橋的地址是：`100.96.2.1/24`

```shell
# 在Node 1上
$ ip route
default via 10.168.0.1 dev eth0
100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.1.0
100.96.1.0/24 dev docker0  proto kernel  scope link  src 100.96.1.1
10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.2
---
# 在Node 2上
$ ip route
default via 10.168.0.1 dev eth0
100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.2.0
100.96.2.0/24 dev docker0  proto kernel  scope link  src 100.96.2.1
10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.3
```

由於目標 IP 為 100.96.2.3，只會匹配到 **flannel0** 的規則，而 flannel0 為 TUN 設備 （Tunnel 設備）。

在 Linux 中，TUN 設備是一種工作在三層（Network Layer）的虛擬網路設備。TUN 設備的功能非常簡單，即：在操作系統內核和用戶應用程序之間傳遞 IP 包。

> 當操作系統將一個 IP 包發送給 **flannel0** 設備之後，**flannel0** 就會把這個 IP 包，交給創建這個設備的應用程序，也就是 Flannel 進程。這是一個從內核態（Linux 操作系統）向用戶態（Flannel 進程）的流動方向。

Flannel有子網的概念，一台宿主機的容器地址屬於同一個子網，所以跨主通信時候可以根據目的地址找到對應的宿主機。

在例子中，Node 1 的子網是 100.96.1.0/24，container-1 的 IP 地址是 100.96.1.2。Node 2 的子網是 100.96.2.0/24，container-2 的 IP 地址是 100.96.2.3。

這些**子網與宿主機的對應關係**，正是**保存在 Etcd** 當中，如下所示：
```shell
$ etcdctl ls /coreos.com/network/subnets
/coreos.com/network/subnets/100.96.1.0-24
/coreos.com/network/subnets/100.96.2.0-24
/coreos.com/network/subnets/100.96.3.0-24s
---
$ etcdctl get /coreos.com/network/subnets/100.96.2.0-24
{"PublicIP":"10.168.0.3"}
```
flanneld 在收到 container-1 發給 container-2 的 IP 包之後，就會把**這個 IP 包直接封裝在一個 UDP 包**里，然後發送給 Node 2。這個 UDP 包的源地址，就是 flanneld 所在的 Node 1 的地址，而目的地址，則是 container-2 所在的宿主機 Node 2 的地址。

> 重要前提， docker0 網橋的地址範圍必須是 Flannel 為宿主機分配的子網
> ```shell
> $ FLANNEL_SUBNET=100.96.1.1/24
> $ dockerd --bip=$FLANNEL_SUBNET ...
> ```

基於 Flannel UDP 跨主機通訊基本原理
![](media/16769084797731/16769085808049.jpg)
 
> Flannel UDP 模式提供的其實是一個三層的 Overlay 網路:
> 1. 對發出端的 IP 包進行 UDP 封裝
> 2. 在接收端進行解封裝拿到原始的 IP 包
> 3. 進而把這個 IP 包轉發給目標容器
> 
> 查詢路由 IP 根據路由表進入 TUN （flannel0） 設備，從而回到用戶進程；再者，flanneld 進行 UDP 封裝和解封裝在用戶態，UDP 傳送又會進入內核態，**上下文切換和用戶態操作，因此效能較差**。

## VXLAN

VXLAN，即 `Virtual Extensible LAN`（虛擬可擴展局域網），是 Linux 內核本身就支持的一種網路虛似化技術。所以說，**VXLAN 可以完全在內核態實現上述封裝和解封裝的工作**，從而通過與前面相似的**隧道機制**，構建出**覆蓋網路（`Overlay Network`）**。

為了能夠在二層網路(數據鏈路層)上打通隧道，VXLAN 會在宿主機上設置一個特殊的網路設備作為隧道的兩端。這個設備就叫作 `VTEP`，即：`VXLAN Tunnel End Point`（虛擬隧道端點）。
> VTEP 是每台宿主機上的 flanneld 維護的。

> VXLAN 設計思想為：在現有的三層網路之上，**覆蓋一層虛擬的、由內核 VXLAN 模塊負責維護的二層網路**，使得連接在這個 VXLAN 二層網路上的主機（虛擬機或者容器都可以）之間，可以像在同一個局域網（LAN）里那樣溝通。

基於 Flannel VXLAN 模式的跨主通信的基本原理
![](media/16769084797731/16771671552883.jpg)

> Outer 封裝為宿主機網路的數據幀
> Inner 封裝為內部 VXLAN 網路的數據幀
> VXLAN Header 裡面有個重要標誌為 `VNI`，**辨識數據幀是哪個 VTEP 設備處理的**。


當一個節點加入 Flannel 網路後，所有節點會添加這個節點的 `flannel.1` 設備（ VTEP 設備）的：
- 路由規則
- ARP 記錄，設備 MAC 與當前節點 IP 關聯關係

> 透過 ARP 記錄，才能找到目的 VTEP 設備的 MAC 地址

再透過 ARP 取得的 MAC 去拿到 `flannel.1` 網橋對應的 `FDB` 訊息（**目的為找 UDP 包要發給哪台宿主機**）
> Linux 內核裡面，網橋設備進行轉發的依據，來自於一個叫作 FDB（`Forwarding Database`）的轉發數據庫
> 
```shell
# Node 2 的資訊
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
...
10.1.16.0       10.1.16.0       255.255.255.0   UG    0      0        0 flannel.1
---
# 在 Node 1 上，加入 ARP 記錄
$ ip neigh show dev flannel.1
10.1.16.0 lladdr 5e:f8:4f:00:e3:37 PERMANENT
---
# 在Node 1上，使用目的 VTEP 設備的 MAC地址 進行查詢
$ bridge fdb show flannel.1 | grep 5e:f8:4f:00:e3:37
5e:f8:4f:00:e3:37 dev flannel.1 dst 10.168.0.3 self permanent
```

Flannel VXLAN 模式的數據幀
![](media/16769084797731/16772054608724.jpg)

## 小結

在進行系統級編程的時候，有一個非常重要的優化原則，就是要**減少用戶態到內核態的切換次數**，並且把**核心的處理邏輯都放在內核態進行**。

VXLAN 模式組建的覆蓋網路，其實是一個由不同宿主機上的 VTEP 設備，也就是 flannel.1 設備組成的虛擬二層網路。

> 如果想要在集群中實踐 Flannel 的話，可以在 Master 節點上執行如下命令來替換網路插件:
> 1. $ rm -rf /etc/cni/net.d/*
> 2. $ kubectl delete -f "https://cloud.weave.works/k8s/net?k8s-version=1.11"
> 3. 在 /etc/kubernetes/manifests/kube-controller-manager.yaml 里，為容器啓動命令添加如下兩個參數：--allocate-node-cidrs=true--cluster-cidr=10.244.0.0/16
> 4. 重啓所有 kubelet
> 5. $ kubectl create -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml。

此文章為2月Day15學習筆記，內容來源於極客時間[《深入剖析Kuberentes》](https://time.geekbang.org/column/article/65287)

《Linux0.11源碼趣讀》第二季重磅上線