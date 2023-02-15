# Day09 深入剖析 Kubernetes - 容器編排與 Kubernetes 作業管理

## 聲明式API與Kubernetes編程範式

命令式 API 操作
- Docker swarm
```shell
$ docker service create --name nginx --replicas 2  nginx
$ docker service update --image nginx:1.7.9 nginx
```
- Kubernetes
    - 先行使用 `kubectl create`
    - 修改 YAML 文件後，使用 `kubectl replace` 進行更新操作
```shell
$ kubectl create -f nginx.yaml
$ kubectl replace -f nginx.yaml
```

聲明式 API 操作

- `kubectl apply` 指令
    - 用戶不用管具體操作是創建，只是更新，統一使用 `apply` 指令操作
    - 修改 YAML 文件後，一樣使用 `kubectl apply` 進行更新操作
```shell
$ kubectl apply -f nginx.yaml
```

- **`PATCH` API，為聲明式 API 最主要的能力**

### kubectl replace vs kubectl apply
兩者差異在於：

- `kubectl replace` 的執行過程，是**使用新的 YAML 文件中的 API 對象，替換原有的 API 對象**
    - 一次處理一個寫請求操作
    
- `kubectl apply`，則是**執行了一個對原有 API 對象的 PATCH 操作**
    - 一次能處理多個寫操作，且具備 `Merge` 能力

### Dynamic Admission Control
> Ref:
> - [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)

在 Kubernetes 中，當一個 Pod 或 API 對象被提交給 API Server 之後，總會有一些**初始化**的工作需要在它們被 Kubernetes **項目正式處理前進行操作**，如**自動為所有 Pod 加入特定標籤（Labels）**。

- **容器注入可以以 `ConfigMap` 的方式保存起来**
    - 如 Istio 將 Envoy 容器定義放在 ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-initializer
data:
  config: |
    containers:
      - name: envoy
        image: lyft/envoy:845747db88f102c0fd262ab234308e9e22f693a1
        command: ["/usr/local/bin/envoy"]
        args:
          - "--concurrency 4"
          - "--config-path /etc/envoy/envoy.json"
          - "--mode serve"
        ports:
          - containerPort: 80
            protocol: TCP
        resources:
          limits:
            cpu: "1000m"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "64Mi"
        volumeMounts:
          - name: envoy-conf
            mountPath: /etc/envoy
    volumes:
      - name: envoy-conf
        configMap:
          name: envoy
```

- 通常會有一個 `Initializer` 的 Pod，來實現**檢查用戶創建的 Pod 是否有被初始化過**
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        app: envoy-initializer
      name: envoy-initializer
    spec:
      containers:
        - name: envoy-initializer
          image: envoy-initializer:0.0.1
          imagePullPolicy: Always
    ```
        
    - 如上 Istio 例子，Initializer 偽代碼
        - 把這個 `ConfigMap` 里存儲的 `containers` 和 `volumes` 字段，直接添加進一個空的 Pod 對象
        - 關鍵點，**Kubernetes API 提供一個方法 `TwoWayMergePatch`，可以直接合併兩個新舊 Pod**
    ```go
    func doSomething(pod) {
      cm := client.Get(ConfigMap, "envoy-initializer")
    
      newPod := Pod{}
      newPod.Spec.Containers = cm.Containers
      newPod.Spec.Volumes = cm.Volumes
    
      // 生成 patch 數據
      patchBytes := strategicpatch.CreateTwoWayMergePatch(pod, newPod)
    
      // 發起 PATCH 請求，修改這個 pod 對象
      client.Patch(pod.Name, patchBytes)
    }  
    ```

- Kubernetes 還允許通過配置，指定對什麼樣的資源做初始化操作
    - 如下面例子，每一個 API v1 的 Pod 都會在 Metadata 上加上 Initialize 字段
    - **如果初始化操作是依有無這個字段，那麼完成初始化操作，需要清除 `metadata.initializers.pending` 字段**
    ```yaml
    apiVersion: admissionregistration.k8s.io/v1alpha1
    kind: InitializerConfiguration
    metadata:
      name: envoy-config
    initializers:
      // 这个名字必须至少包括两个 "."
      - name: envoy.initializer.kubernetes.io
        rules:
          - apiGroups:
              - "" // 前面说过， ""就是core API Group的意思
            apiVersions:
              - v1
            resources:
              - pods
    ---
    # result demo
    apiVersion: v1
    kind: Pod
    metadata:
      initializers: # 初始化加入的字段
        pending:
          - name: envoy.initializer.kubernetes.io
      name: myapp-pod
      labels:
        app: myapp
    ...
    ```    
    - 另一個配置方式，為**在具體 Pod 的 `Annotation` 屬性加入要使用的 `Initializer`**
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata
      annotations: # here
        "initializer.kubernetes.io/envoy": "true"
        ...
    ```

## 小結

以 Istio 項目為例，說明如何使用 Kubernetes 的 `Initializer` 特性，完成 Envoy **容器自動注入**的原理

>  Istio 現在是 `admission hook`。這部分功能變化太多，已經不是前面提到的 Initializer 方式，最好是自己寫個 sidecar operator 來管理

- 聲明式 API，指的就是只需要提交一個定義好的 API 對象來聲明使用，所期望的狀態是什麼樣子
- 聲明式 API 允許有多個 API 寫入端，以 `PATCH` 的方式對 API 對象進行修改，而無需關心本地原始 YAML 文件的內容
- 最重要的是，有了上述兩個能力，Kubernetes 項目才可以基於對 API 對象的增、刪、改、查，在完全無需外界干預的情況下，完成對**實際狀態**和**期望狀態**的調諧（`Reconcile`）過程（或是叫 `Sync Loop`（同步循環））。


此文章為2月Day09學習筆記，內容來源於極客時間[《深入剖析Kuberentes》](https://time.geekbang.org/column/article/41769)