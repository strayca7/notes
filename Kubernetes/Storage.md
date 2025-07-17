# ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  labels:
    app: my-app
data:
  # 类属性键；每一个键都映射到一个简单的值
  log-level: "info"
  app-name: "my-app"

  # 类文件键
  app.properties: |
    server.port=8080
    app.name=my-app
    log.level=info
```

`data` 字段中所有键值对，如果不在 Pod 中用 `items` 进行 key-values 映射，默认会将 ConfigMap 中的所有 key 全部转化为一个个同名的文件进行挂载。



## 热更新

> [!WARNING]
>
> 环境变量（`env` 或 `envFrom`）和通过 `subPath` 挂载的单个文件不支持热更新。环境变量仅在 Pod 启动时注入，后续变更需重启 Pod；使用`subPath` kubernetes 会将文件直接绑定到容器中，不使用符号链接。

**Volume** 挂载到容器内时，虽然底层挂载方式是基于 **bind mount**，但为了实现热更新，Kubernetes 使用了**符号链接（symlink）**来管理和切换配置文件。

当更新 ConfigMap 时，Kubernetes 会使用更新后的数据创建一组新文件，并更新符号链接以指向这些新文件。此过程设计为原子过程，新的配置文件写完成后才更新符号链接，这样可确保应用程序始终读取一组一致的配置数据，无需重启容器，默认同步周期约 1 分钟（可通过 `kubelet --sync-frequency` 调整）。



更新 ConfigMap 目前不会触发相关的 Pod 滚动更新，但是可以修改 Pod annotations 的方式前置触发滚动更新：

```bash
kubectl patch deploy nginx --patch'{"spec":{"template":{"metadata":{"annotations":{"version/config":"任意"}}}}}'
```

需要注意，只有 `version/config` 的值发生改变时 ，才会触发 Deployment 的滚动更新，触发过一次滚动更新后，如果字段值没有改变，将不会再次触发。

> 普通 Pod 不支持，只有在 Deployment 下才能生效。
>
> annotations 为 `map[string]string` 类型。









































