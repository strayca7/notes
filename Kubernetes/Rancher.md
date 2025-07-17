查看 rancher 根据 helm chart 创建的 app（helm release）

```bash
kubectl get apps.catalog.cattle.io -A
```

这相当于：

```bash
helm list
```

