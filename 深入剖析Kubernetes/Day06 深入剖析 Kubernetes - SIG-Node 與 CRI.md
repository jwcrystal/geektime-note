# Day06 深入剖析 Kubernetes - SIG-Node 與 CRI

## SIG-Node 與 CRI

`kubelet` 主要功能：在調度這一步完成後，kubelet 需要負責將這個調度成功的 Pod，在宿主機上創建出來，並把它所定義的各個容器啓動起來。

![](media/16760990687688/16761004778338.png)


