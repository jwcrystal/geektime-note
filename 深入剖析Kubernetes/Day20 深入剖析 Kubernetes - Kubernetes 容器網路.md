# Day20 深入剖析 Kubernetes - Kubernetes 容器網路

## 從外界連通 Service 與 Service 調試三板斧

Service 的訪問入口，其實就是每台宿主機上由 kube-proxy 生成的 iptables 規則，以及 kube-dns 生成的 DNS 記錄。
> Service 的訪問訊息在 Kubernetes 集群外，是無效的

**Kubernetes 提供三種方式：NodePort、LoadBalancer、External Name**

### NodePort

以下面 YAML 為例
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - nodePort: 8080
    targetPort: 80
    protocol: TCP
    name: http
  - nodePort: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
```

不顯式地聲明 `nodePort` 字段，Kubernetes 就會為你分配隨機的可用端口來設置代理。這個端口的範圍默認是 `30000-32767`，你 **可以通過 kube-apiserver 的 `–service-node-port-range` 參數來修改它**。

要訪問 Service，透過聲明的端口即可訪問
```shell
<任何一台宿主機的IP地址>:8080
---
# kube-proxy 在每台宿主機建立下面的 iptabels 規則
# KUBE-SVC-67RL4FN6JRUPOJYM: 跟前一章敘述的一樣，為一組隨機規則集合
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx: nodePort" -m tcp --dport 8080 -j KUBE-SVC-67RL4FN6JRUPOJYM
```
留意的點，為在 NodePort 模式，Kubernetes 會在 IP 包離開宿主機後，對 IP 包做一次 `SNAT` 操作，如下所示
```shell
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

為什麼需要對流出的 IP 包做 SNAT 操作？

原理如下所示
```shell
       client
             \ ^
              \ \
               v \
   node 1 <--- node 2
    | ^   SNAT
    | |   --->
    v |
 endpoints
```

如果 Node 2 的負載均衡規則轉發給 Node 1 上的 Pod 處理請求，處理後如果**沒有實現 SNAT 操作，會直接從 Node 1 返回給 client**。可能會造成 client 處理判斷錯誤。

上面例子，**在 Pod 需要知道所有請求來源是不可行的**，這時候需要將 Service 的 `spec.externalTrafficPolicy` 字段設置為 `local`，這就**保證了所有 Pod 通過 Service 收到請求之後，一定可以看到真正的、外部 client 的源地址。**

```shell
       client
       ^ /   \
      / /     \
     / v       X
   node 1     node 2
    ^ |
    | |
    | v
 endpoint
```
> 一台宿主機上的 iptables 規則，會設置為只將 IP 包轉發給運行在這台宿主機上的 Pod。所以這時候，Pod 可以直接使用源地址將回復包發出，不需要事先進行 SNAT 了。

### LoadBalancer

以下面 YAML 為例
```yaml
kind: Service
apiVersion: v1
metadata:
  name: example-service
spec:
  ports:
  - port: 8765
    targetPort: 9376
  selector:
    app: example
  type: LoadBalancers
```
適用於公有雲。
在公有雲 Kubernetes 中，會調用 `CloudProvider` 在公有雲上創建一個負載均衡服務，並且把被代理的 Pod 的 IP 地址配置給負載均衡服務做後端。

### ExternalName

以下面 YAML 為例
```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: my.database.example.com
---
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  externalIPs:
  - 80.11.12.10
```
ExternalName 類型的 Service，其實是在 `kube-dns` 里添加了一條 `CNAME` 記錄。這時，訪問 `my-service.default.svc.cluster.local` 就和訪問 `my.database.example.com` 這個域名是一個效果。

而指定 externalIPs=80.11.12.10，就可以通過訪問 80.11.12.10:80 訪問到被代理的 Pod。

## Service 相關問題

**Service 相關的問題，其實都可以通過分析 Service 在宿主機上對應的 iptables 規則（或者 IPVS 配置）得到解決。**

- 當你的 Service 沒辦法通過 DNS 訪問到的時候。你就需要區分到底是 Service 本身的配置問題，還是集群的 DNS 出了問題。一個行之有效的方法，就是檢查 Kubernetes 自己的 Master 節點的 Service DNS 是否正常：
```shell
# 在一個Pod里執行
$ nslookup kubernetes.default
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local
```
> 如果訪問 `kubernetes.default` 返回的值都有問題，那就需要檢查 `kube-dns` 的運行狀態和日誌了。否則的話，應該檢查 Service 定義是不是有問題。

- 如果 Service 沒辦法通過 ClusterIP 訪問到的時候，首先應該檢查的是這個 Service 是否有 Endpoints：
```shell
$ kubectl get endpoints hostnames
NAME        ENDPOINTS
hostnames   10.244.0.5:9376,10.244.0.6:9376,10.244.0.7:9376
```
> 如果 Pod 的 readniessProbe 沒通過，它也不會出現在 Endpoints 列表裡

- 如果 Endpoints 正常，那麼需要確認 `kube-proxy` 是否在正確運行。在我們通過 `kubeadm` 部署的集群里，應該會看到 `kube-proxy` 輸出的日誌如下所示:
```shell
I1027 22:14:53.995134    5063 server.go:200] Running in resource-only container "/kube-proxy"
I1027 22:14:53.998163    5063 server.go:247] Using iptables Proxier.
I1027 22:14:53.999055    5063 server.go:255] Tearing down userspace rules. Errors here are acceptable.
I1027 22:14:54.038140    5063 proxier.go:352] Setting endpoints for "kube-system/kube-dns:dns-tcp" to [10.244.1.3:53]
I1027 22:14:54.038164    5063 proxier.go:352] Setting endpoints for "kube-system/kube-dns:dns" to [10.244.1.3:53]
I1027 22:14:54.038209    5063 proxier.go:352] Setting endpoints for "default/kubernetes:https" to [10.240.0.2:443]
I1027 22:14:54.038238    5063 proxier.go:429] Not syncing iptables until Services and Endpoints have been received from master
I1027 22:14:54.040048    5063 proxier.go:294] Adding new service "default/kubernetes:https" at 10.0.0.1:443/TCP
I1027 22:14:54.040154    5063 proxier.go:294] Adding new service "kube-system/kube-dns:dns" at 10.0.0.10:53/UDP
I1027 22:14:54.040223    5063 proxier.go:294] Adding new service "kube-system/kube-dns:dns-tcp" at 10.0.0.10:53/TCP
```
> 如果 kube-proxy 一切正常，就需要應該仔細查看宿主機上的 iptables 規則。

一個 iptables 模式的 Service 對應的規則，包括：

- `KUBE-SERVICES` 或者 `KUBE-NODEPORTS` 規則對應的 `Service` 的入口鏈，這個規則應該與 VIP 和 Service 端口一一對應
- `KUBE-SEP-(hash)` 規則對應的 `DNAT 鏈，這些規則應該與 `Endpoints` 一一對應
- `KUBE-SVC-(hash)` 規則對應的**負載均衡鏈**，這些規則的數目應該與 `Endpoints` 數目一致
- 如果是 `NodePort` 模式的話，還有 `POSTROUTING` 處的 `SNAT` 鏈

另一種典型問題，為 **Pod 無法透過 Service 訪問自己**。
```shell
# hairpin-veth 模式
$ for d in /sys/devices/virtual/net/cni0/brif/veth*/hairpin_mode; do echo "$d = $(cat $d)"; done
/sys/devices/virtual/net/cni0/brif/veth4bfbfe74/hairpin_mode = 1
/sys/devices/virtual/net/cni0/brif/vethfc2a18c5/hairpin_mode = 1
---
# promiscuous-bridge 模式
$ ifconfig cni0 |grep PROMISC
UP BROADCAST RUNNING PROMISC MULTICAST  MTU:1460  Metric:1
```

> kubelet 的 `hairpin-mode` 沒有被正確設置，需要確保將 kubelet 的 hairpin-mode 設置為 `hairpin-veth` 或者 `promiscuous-bridge` 即可。


通過查看這些鏈的數量、轉發目的地址、端口、過濾條件等訊息，就能容易發現一些異常的蛛絲馬跡。

## 小結

討論了從外部訪問 Service 的三種方式，和具體的工作原理：
- NodePort
- LoadBalancer
- External Name

Service，其實就是 Kubernetes 為 Pod 分配的、固定的、基於 iptables（或者 IPVS）的訪問入口。而這些訪問入口代理的 Pod 訊息，則來自於 Etcd，由 kube-proxy 通過控制循環來維護。

此文章為2月Day20學習筆記，內容來源於極客時間[《深入剖析Kuberentes》](https://time.geekbang.org/column/article/68964)

《Linux0.11源碼趣讀》第二季重磅上線