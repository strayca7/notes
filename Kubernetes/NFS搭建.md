```bash
# 在 10.3.10.16 上执行（非集群节点）
yum install -y nfs-utils
mkdir -p /data/nfs_share
echo "/data/nfs_share *(rw,sync,no_root_squash,no_subtree_check)" > /etc/exports

#/data/nfs_share：服务器上要共享的目录路径。
# *：允许所有客户端访问（也可指定 IP 或网段，如 192.168.1.0/24）。
# (rw,sync,no_root_squash)：共享选项：
    # rw：客户端具有读写权限（默认是只读 ro）。
    # sync：数据同步写入磁盘（保证一致性，但性能较低；异步模式为 async）。
    # no_root_squash：客户端的 root 用户在访问时保留 root 权限​（默认会映射为匿名用户，安全风险较高，需谨慎使用）。

systemctl enable --now nfs-server （或者重启）

#验证共享目录
showmount -e localhost
# 预期输出：（如果出现非预期结果，请先根据输出结果自行判断为什么问题）
# /data/nfs_share *
```

**在所有K8s节点安装NFS客户端**

```bash
# 在master01、node01、node02上分别执行：
sudo yum install -y nfs-utils

# 验证客户端可用性（在任意节点执行）
showmount -e 10.3.10.16
# 预期输出：（如果出现非预期结果，请先根据输出结果自行判断为什么问题）
# /data/nfs_share *
```

**创建 StorageClass**

```bash
# 3.1 添加Helm仓库
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

# 3.2 创建专用命名空间
kubectl create ns nfs-provisioner

# 3.3 使用Helm部署Provisioner
helm upgrade --install nfs-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-provisioner \
  --set nfs.server=10.3.10.16 \
  --set nfs.path=/data/nfs_share \
  --set image.repository=swr.cn-north-4.myhuaweicloud.com/ddn-k8s/k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner \
  --set image.tag=v4.0.2 \
  --set storageClass.name=nfs-storage \
  --set storageClass.onDelete="retain" \
  --set extraArgs.enableFixPath=true \
  --set storageClass.defaultClass=true

# 关键参数说明：
# nfs.server: NFS服务器IP
# nfs.path: 共享目录路径
# storageClass.name: 存储类名称
# storageClass.defaultClass: 设为默认存储类
# nfs-storageclass.yaml


kubectl get pod -n nfs-provisioner -w
# 预期输出：
# NAME                                                              READY   STATUS    RESTARTS   AGE
# nfs-provisioner-nfs-subdir-external-provisioner-7d88f5d58-xxxxx   1/1     Running   0          30s

kubectl get storageclass
# 确认 nfs-storage 显示为 (default)