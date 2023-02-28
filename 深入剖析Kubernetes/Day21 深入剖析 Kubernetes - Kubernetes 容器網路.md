# Day21 深入剖析 Kubernetes - Kubernetes 容器網路

## 談談Service與Ingress

這種全局的、為了代理不同後端 Service 而設置的負載均衡服務，就是 Kubernetes 里的 Ingress 服務。
> Ingress，就是 Service 的 Service

Ingress 定義如下所示
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```

**IngressRule 的 Key，就叫做 `host`。它必須是一個標準的域名格式（`Fully Qualified Domain Name`）的字符串，而不能是 IP 地址。**

**`path` 則對應 `Deployment` 的 `Service`。**

- /tea：tea-svc
- /coffee： coffee-svc

> FQDN，可以參考 **RFC 3986** 規範格式

有了 `Ingress` 這樣一個統一的抽象，Kubernetes 用戶就無需關心 Ingress 的具體細節了。

而 `Ingress Controller` 會根據定義的 Ingress 對象，提供對應的代理能力。目前，業界常用的各種反向代理項目，如下面常見項目所示：
- Nginx
- HAProxy
- Envoy
- Traefik 

都已經為 Kubernetes 專門維護了對應的 Ingress Controller。

### Nginx Ingress Controller

Nginx Ingress Controller 定義如下所示

- 一個監聽 Ingress 對象以及它所代理的後端 Service 變化的控制器。
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        ...
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.20.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
            - name: http
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
```

當一個新的 Ingress 對象由用戶創建後，`nginx-ingress-controller` 就會根據 Ingress 對象里定義的內容，生成一份對應的 Nginx 配置文件（`/etc/nginx/nginx.conf`），並使用這個配置文件啓動一個 Nginx 服務。

> 如果這裡只是被代理的 Service 對象被更新，nginx-ingress-controller 所管理的 Nginx 服務是不需要重新加載（reload）的。這當然是因為 **nginx-ingress-controller 通過 `Nginx Lua` 方案實現了 Nginx Upstream 的動態配置**。

為了讓用戶使用這個 Nginx 服務，還需要創建一個 Service 把 Nginx 暴露出去。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

```shells
$ kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.105.72.96   <none>        80:30044/TCP,443:31453/TCP   3h
```
**這個 Service 唯一工作，就是將所有攜帶 ingress-nginx 標籤的 Pod 的 80 和 433 端口暴露出去**。
> 而如果是在公有雲上，則需要暴露 LoadBalancer 類型的 Service。

### Demo : Coffee Ingress
> Ref:
> - [Nginx Ingress Controller Example](https://github.com/resouer/kubernetes-ingress/tree/master/examples/complete-example)

當 Ingress Controller 和相對應的 Service 部署完後，即可部署 demo 文件，可以參考 Reference 內的 **coffee** 文件

部署完成，如下所示：
```shell
# 方便展示，設置為環境變數
$ IC_IP=10.168.0.2 # 任意一台宿主機的地址
$ IC_HTTPS_PORT=31453 # NodePort端口
---
$ kubectl get ingress
NAME           HOSTS              ADDRESS   PORTS     AGE
cafe-ingress   cafe.example.com             80, 443   2h

$ kubectl describe ingress cafe-ingress
Name:             cafe-ingress
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<none>)
TLS:
  cafe-secret terminates cafe.example.com
Rules:
  Host              Path  Backends
  ----              ----  --------
  cafe.example.com  
                    /tea      tea-svc:80 (<none>)
                    /coffee   coffee-svc:80 (<none>)
Annotations:
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  4m    nginx-ingress-controller  Ingress default/cafe-ingress
```
>  Ingress 對象最核心的部分，正是 Rules 字段（即為轉發規則）。

即可透過訪問 Ingress 地址和端口，訪問到前面部署的應用
```shell
$ curl --resolve cafe.example.com:$IC_HTTPS_PORT:$IC_IP https://cafe.example.com:$IC_HTTPS_PORT/coffee --insecureServer address: 10.244.1.56:80
Server name: coffee-7dbb5795f6-vglbv
Date: 03/Nov/2018:03:55:32 +0000
URI: /coffee
Request ID: e487e672673195c573147134167cf898
```

Ingress Controller 也允許通過 Pod 啓動命令里的 `–default-backend-service` 參數，設置一條默認規則
- –default-backend-service=nginx-default-backend

> 也就是說可以自定義失敗 404 頁面等等的請求

## 小結

**目前 Kubernetes Ingress 只能工作在七層，而 Service 只能工作在四層。**

為應用進行 TLS 配置等 HTTP 相關的操作時，都必須通過 Ingress 來進行。

可以透過社區的 Ingress Controller 或是自定義 Ingress Controller 處理前面提到的問題，如 `Nginx Ingress Controller`。

此文章為2月Day21學習筆記，內容來源於極客時間[《深入剖析Kuberentes》](https://time.geekbang.org/column/article/69214)

《Linux0.11源碼趣讀》第二季重磅上線