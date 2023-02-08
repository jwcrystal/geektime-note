# Day03 深入剖析 Kubernetes - Kubernetes 默認調度器策略解析

## Kubernetes 默認調度器策略解析

- `Predicate` 在調度過程中，作用為 **filter**

默認調度策略有 4 種：

- `GeneralPredicates`:
- `跟 volume 相關過濾規則`:
- `跟宿主機 相關過濾規則`:
- `跟 Pod 相關過濾規則`: