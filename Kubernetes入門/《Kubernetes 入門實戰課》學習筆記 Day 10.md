# 《Kubernetes 入門實戰課》學習筆記 Day 10

## Kubernetes 技術要點回顧

Kubernetes 是雲原生時代的操作系統，它能夠管理大量節點構成的集群，讓計算資源池化，從而能夠自動地調度運維各種形式的應用。**搭建多節點的 Kubernetes 集群是一件頗具挑戰性的工作**，好在社區里及時出現了 kubeadm 這樣的工具，可以一鍵操作，使用 `kubeadm init`、`kubeadm join` 等命令從無到有地搭建出生產級別的集群。

kubeadm 使用容器技術封裝了 Kubernetes 組件，所以只要節點上安裝了容器運行時（Docker、containerd 等），它就可以自動從網上拉取鏡像，然後以容器的方式運行組件，非常方便。

在實際生產環境的 Kubernetes 集群里，我們學習了 `Deployment`、`DaemonSet`、`Service`、`Ingress`、`Ingress Controller` 等 API 對象:

- [（18 講）](https://time.geekbang.org/column/article/535209)Deployment 是用來管理 Pod 的一種對象，它代表了運維工作中最常見的一類在線業務，在集群中部署應用的多個實例，而且可以很容易地增加或者減少實例數量，從容應對流量壓力。Deployment 的定義里有兩個關鍵字段：一個是 **replicas**，它指定了實例的數量；另一個是 **selector**，它的作用是使用標籤篩選出被 Deployment 管理的 Pod，這是一種非常靈活的關聯機制，實現了 API 對象之間的松耦合關係
- [（19 講）](https://time.geekbang.org/column/article/536803)DaemonSet 是另一種部署在線業務的方式，它很類似 Deployment，但會在集群里的每一個節點上運行一個 Pod 實例，類似 Linux 系統里的守護進程，適合日誌、監控等類型的應用。DaemonSet 能夠任意部署 Pod 的關鍵概念是污點（taint）和容忍度（toleration）。Node 會有各種污點，而 Pod 可以使用容忍度來忽略污點，合理使用這兩個概念就可以調整 Pod 在集群里的部署策略
- [（20 講）](https://time.geekbang.org/column/article/536829)由 Deployment 和 DaemonSet 部署的 Pod，在集群中處於動態平衡的狀態，總數量保持恆定，但也有臨時銷毀重建的可能，所以 IP 地址是變化的，這就為微服務等應用架構帶來了麻煩。**Service 是對 Pod IP 地址的抽象**，它擁有一個固定的 IP 地址，再使用 iptables 規則把流量負載均衡到後面的 Pod，節點上的 kube-proxy 組件會實時維護被代理的 Pod 狀態，保證 Service 只會轉發給健康的 Pod。Service 還基於 DNS 插件支持域名，所以客戶端就不再需要關心 Pod 的具體情況，只要通過 Service 這個穩定的中間層，就能夠訪問到 Pod 提供的服務
- [（21 講）](https://time.geekbang.org/column/article/538760)Service 是四層的負載均衡，但現在的絕大多數應用都是 HTTP/HTTPS 協議，要實現七層的負載均衡就要使用 Ingress 對象。

Ingress 定義了基於 HTTP 協議的路由規則，但要讓規則生效，還需要 Ingress Controller 和 Ingress Class 來配合工作。
- Ingress Controller 是真正的集群入口，應用 Ingress 規則調度、分發流量，此外還能夠扮演反向代理的角色，提供安全防護、TLS 卸載等更多功能
- **Ingress Class 是用來管理 Ingress 和 Ingress Controller 的概念，方便我們分組路由規則，降低維護成本**。

**不過 Ingress Controller 本身也是一個 Pod，想要把服務暴露到集群外部還是要依靠 Service**。Service 支持 NodePort、LoadBalancer 等方式，但 NodePort 的端口範圍有限，LoadBalancer 又依賴於雲服務廠商，都不是很靈活。**折中的辦法是用少量 NodePort 暴露 Ingress Controller，用 Ingress 路由到內部服務，外部再用反向代理或者 LoadBalancer 把流量引進來**。

### 小結
![image](https://user-images.githubusercontent.com/121911854/214604238-269d171f-22e8-49c0-85a9-1bb365dd8d1f.jpeg)
