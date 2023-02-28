# Day16 深入剖析 Kubernetes - Kubernetes 容器網路

## Kubernetes網路模型與CNI網路插件

從前面例子，可以觀察到有一個共同點，那就是用戶的容器都連接在 `docker0` 網橋上。而網路插件則在宿主機上創建了一個特殊的設備（**UDP 模式創建的是 TUN 設備，VXLAN 模式創建的則是 VTEP 設備**），docker0 與這個設備之間，通過 **IP 轉發（路由表）進行協作**。

Kubernetes 是通過一個叫作 CNI 的接口，維護了一個單獨的網橋來代替 `docker0`。這個網橋的名字就叫作：**CNI 網橋**，它在宿主機上的設備名稱默認是：`cni0`。

> 網路插件真正要做的事情，則是通過某種方法，把**不同宿主機上的特殊設備連通，從而達到容器跨主機通信的目的**。

docker0 換成 cni0 示意圖
- flannel 分配網段 `10.168.0.0/16`
    - 可以透過集群建立時，指定網段
    - 在部署完成後，通過修改 `kube-controller-manager` 的配置文件來指定
```shell
$ kubeadm init --pod-network-cidr=10.244.0.0/16
```
![](media/16770761752031/16775121559462.jpg)

路由表
```shell
# 在 Node 1 上
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
...
10.244.0.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.1.0      10.244.1.0      255.255.255.0   UG    0      0        0 flannel.1
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
```
> CNI 網橋只是接管所有 CNI 插件負責的、即 Kubernetes 創建的容器（Pod）

Kubernetes 之所以要設置這樣一個與 docker0 網橋功能幾乎一樣的 CNI 網橋，主要原因包括兩個方面：

- Kubernetes 項目並沒有使用 Docker 的網路模型（CNM），所以它並不希望、也不具備配置 docker0 網橋的能力
- 與 Kubernetes 如何配置 Pod，也就是 Infra 容器的 Network Namespace 密切相關

CNI 的設計思想，為 Kubernetes 在啓動 Infra 容器之後，就可以直接調用 CNI 網路插件，為這個 Infra 容器的 Network Namespace，配置符合預期的網路棧。

> 一個 Network Namespace 的網路棧包括：網卡（Network Interface）、回環設備（Loopback Device）、路由表（Routing Table）和 iptables 規則。

在宿主機的 `/opt/cni/bin` 目錄下，可以看安裝的執行文件
```shell
$ ls -al /opt/cni/bin/
total 73088
-rwxr-xr-x 1 root root  3890407 Aug 17  2017 bridge
-rwxr-xr-x 1 root root  9921982 Aug 17  2017 dhcp
-rwxr-xr-x 1 root root  2814104 Aug 17  2017 flannel
-rwxr-xr-x 1 root root  2991965 Aug 17  2017 host-local
-rwxr-xr-x 1 root root  3475802 Aug 17  2017 ipvlan
-rwxr-xr-x 1 root root  3026388 Aug 17  2017 loopback
-rwxr-xr-x 1 root root  3520724 Aug 17  2017 macvlan
-rwxr-xr-x 1 root root  3470464 Aug 17  2017 portmap
-rwxr-xr-x 1 root root  3877986 Aug 17  2017 ptp
-rwxr-xr-x 1 root root  2605279 Aug 17  2017 sample
-rwxr-xr-x 1 root root  2808402 Aug 17  2017 tuning
-rwxr-xr-x 1 root root  3475750 Aug 17  2017 vlan
```

CNI 的基礎可執行文件，按照功能可以分為三類：

1. `Main` 插件，它是用來創建具體網路設備的二進制文件
    - bridge（網橋設備）
    - ipvlan
    - loopback（lo 設備）
    - macvlan
    - ptp（Veth Pair 設備）
    - vlan
2. `IPAM`（IP Address Management）插件，它是負責分配 IP 地址的二進制文件
    - dhcp：向 DHCP 服務器請求 IP
    - host-loacl：使用預先配置的 IP 地址進行分配
3. CNI 社區維護的內置 `CNI` 插件
    - flannel：專門為 Flannel 項目提供的 CNI 插件
    - tuning：一個通過 sysctl 調整網路設備參數的二進制文件
    - portmap：一個通過 iptables 配置端口映射的二進制文件
    - bandwidth：一個使用 Token Bucket Filter (TBF) 來進行限流的二進制文件

如 Flannel CNI 配置文件如下所示
- `Delegate` 字段，為這個 CNI 插件並不會自己做事，而是會調用 Delegate 指定的某種 CNI 內置插件來完成
    - 這邊指的就是 **CNI Bridge 插件**
```shell
$ cat /etc/cni/net.d/10-flannel.conflist 
{
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

> Kubernetes 中，處理容器網路相關的邏輯並不會在 kubelet 主幹代碼里執行，而是會在具體的 CRI（Container Runtime Interface，容器運行時接口）實現里完成。
> 
> **ADD** 和 **DEL** 操作，就是 CNI 插件唯一需要實現的兩個方法。

經過 Flannel CNI 插件補充後的、完整的 Delegate 字段如下所示
```yaml
{
    "hairpinMode":true,
    "ipMasq":false,
    "ipam":{
        "routes":[
            {
                "dst":"10.244.0.0/16"
            }
        ],
        "subnet":"10.244.1.0/24",
        "type":"host-local"
    },
    "isDefaultGateway":true,
    "isGateway":true,
    "mtu":1410,
    "name":"cbr0",
    "type":"bridge"
}
```
> Flannel 插件要在 CNI 配置文件里聲明 **hairpinMode=true**。這樣，將來這個集群里的 Pod 才可以通過它自己的 Service 訪問到自己。


## 小結

Kubernetes 中 CNI 網路的實現原理，其實就是直接通過 IP 溝通，把不同宿主機上的特殊設備連通，從而達到容器跨主機通信的目的。

- 所有容器都可以直接使用 IP 地址與其他容器通信，而無需使用 NAT
- 所有宿主機都可以直接使用 IP 地址與所有容器通信，而無需使用 NAT
- 容器自己看到的自己的 IP 地址，和別人（宿主機或者容器）看到的地址是完全一樣的

此文章為2月Day16學習筆記，內容來源於極客時間[《深入剖析Kuberentes》](https://time.geekbang.org/column/article/67351)

《Linux0.11源碼趣讀》第二季重磅上線