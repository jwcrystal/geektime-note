# Day12 深入剖析 Kubernetes - 容器編排與 Kubernetes 作業管理

## 基於角色的權限控制：RBAC
> Ref：
> - [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

Kubernetes 項目中，負責完成授權（Authorization）工作的機制，就是 `RBAC`：**基於角色的訪問控制**（`Role-Based Access Control`）。

三個基本概念：

- `Role`：角色，它其實是一組規則，定義了一組對 Kubernetes API 對象的操作權限
- `Subject`：被作用者，既可以是人，也可以是機器，也可以是你在 Kubernetes 里定義的用戶
- `RoleBinding`：定義了被作用者和角色的綁定關係

Role、RoleBinding 本身也是 Kubernetes API 對象
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace # 只在 Namespace 有效
  name: example-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace # 只在 Namespace 有效
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```
> **Namespace 是 Kubernetes 項目里的一個邏輯管理單位**。不同 Namespace 的 API 對象，在通過 kubectl 命令進行操作的時候，是互相隔離開的。

上面提到的 `User`，只是授權系統中的邏輯概念，**需要通過外部認證服務**，如 KeyStone。大部分私有環境下，只需要採用 Kubernetes 內置的用戶（`Service Account`）即可

若非命名空間中的對象（如 Node 等等）要存取，只能使用 `ClusterRole` 和 `ClusterRoleBinding`。與前者差異只在於沒有 Namespace 字段。
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrolebinding
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: example-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

Role 權限與 Rules 字段細化
- 如下示意，rule 只針對 my-config，且只有讀取權限
```yaml
verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
```

前面提到負責管理內置用戶的，為 `Service Account`，簡稱為 `SA`
- 定義很簡單，只需要兩個基本字段： Name、Namespace
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: mynamespace
  name: example-sa
```

把上面 RoleBiding 定義改為 SA，示意如下
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: ServiceAccount
  name: example-sa
  namespace: mynamespace
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```
另外，可以觀察到，**創建後，Kubernetes 自動為 SA 建立一組 secret 並綁定**
```yaml
$ kubectl get sa -n mynamespace -o yaml
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    creationTimestamp: 2018-09-08T12:59:17Z
    name: example-sa
    namespace: mynamespace
    resourceVersion: "409327"
    ...
  secrets:
  - name: example-sa-token-vmfg6 # 自動建立綁定
```

當有 Pod 使用到創建的 SA，我們也可以觀察到其中 secret 對象（`token`）也會自動被掛載到容器目錄(**/var/run/secrets/kubernetes.io/serviceaccount**)下，如下所示
```yaml
$ kubectl describe pod sa-token-test -n mynamespace
Name:               sa-token-test
Namespace:          mynamespace
...
Containers:
  nginx:
    ...
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from example-sa-token-vmfg6 (ro)
---
$ kubectl exec -it sa-token-test -n mynamespace -- /bin/bash
root@sa-token-test:/# ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt namespace  token
```

另外要提起的一點為，Pod 沒有指定 SA 時，Kubernetes 會默認配置名為 `default` 的 SA，而這個**默認 SA 是沒有關聯綁定 Role**，也就是說會擁有大部分的操作權限。
```yaml

$kubectl describe sa default
Name:                default
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-s8rbq
Tokens:              default-token-s8rbq
Events:              <none>

$ kubectl get secret
NAME                  TYPE                                  DATA      AGE
kubernetes.io/service-account-token   3         82d

$ kubectl describe secret default-token-s8rbq
Name:         default-token-s8rbq
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=default
              kubernetes.io/service-account.uid=ffcb12b2-917f-11e8-abde-42010aa80002

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      <TOKEN>
```

> **在生產環境中，建議為所有 Namespace 下的默認 ServiceAccount，綁定一個只讀權限的 Role**，避免一時疏忽而導致 Kuberentes 集群 crash。


除了用戶 `User`，Kubernetes 還有用戶組（`Group`）的概念，也就是一組用戶，而這概念也適用於在 SA。

實際上，一個 ServiceAccount，在 Kubernetes 里對應的 User 的名字是：
- 用戶 User
```yaml
system:serviceaccount:<Namespace名字>:<ServiceAccount名字>
```
- 用戶組 Group
```yaml
system:serviceaccounts:<Namespace名字>
```

如下例子，作用範圍為
- mynamespace 命名空間下的所有 SA
```yaml
subjects:
- kind: Group
  name: system:serviceaccounts:mynamespace
  apiGroup: rbac.authorization.k8s.io
```
- 系統內所有的 SA
```yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```
> 在 Kubernetes 中已經內置了很多個為系統保留的 `ClusterRole`，它們的名字都以 `system:` 開頭。
>
> 也提供預先定義幾組 ClusterRole：
> - **cluster-admin** （最高權限）
> - admin
> - edit
> - view

```yaml
$ kubectl describe clusterrole cluster-admin -n kube-system
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate=true
PolicyRule:
  Resources  Non-Resource URLs Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
```

## 小結

所謂**角色（Role），其實就是一組權限規則列表**。而我們分配這些權限的方式，就是**通過創建 `RoleBinding` 對象，將被作用者（subject）和權限列表進行綁定**。

與之對應的 `ClusterRole` 和 `ClusterRoleBinding`，則是 Kubernetes 集群級別的 Role 和 RoleBinding，**它們的作用範圍不受 Namespace 限制**。

儘管權限的被作用者可以有很多種（如 User、Group 等），但在我們平常的使用中，**最普遍的用法還是 ServiceAccount**。因此，`Role + RoleBinding + ServiceAccount` 的權限分配方式是要重點掌握的內容。

此文章為2月Day12學習筆記，內容來源於極客時間[《深入剖析Kuberentes》](https://time.geekbang.org/column/article/42154)

《Linux0.11源碼趣讀》第二季重磅上線